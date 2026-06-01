# Operations

Day-2 stuff: adding nodes, debugging, monitoring, tuning. Assumes the
panel is already up (see [deployment.md](deployment.md)).

## The admin UI tour

`https://<panel>/admin` after sign-in.

- **Dashboard** — system stats, recent activity, quick-start checklist.
- **Infrastructure / Nodes** — list of exit nodes; **Add node manually**
  for an already-set-up box, **Add node via SSH** for fresh VPS.
- **Infrastructure / Add node** — auto-provisioning page, streams live
  bootstrap logs.
- **Access / Plans** — traffic + duration tiers (0 days = "never expires",
  0 GB = "unlimited").
- **Access / Users** — list, create, edit, disable, delete. Role is
  `user` or `admin`.
- **Access / Subscriptions** — one per user×plan pair. Holds the unique
  subscription URL token, traffic counter, expiry. Online dot lights
  green when `last_seen_at` was updated within the current minute.

## Daily ops cheatsheet

### Add a user + give them VPN access

1. Plans page → create a plan if you don't have one.
2. Users page → "Add user" with email + temporary password.
3. Subscriptions page → "Add subscription" → pick user + plan.
4. The subscription row appears with a Copy button next to the token.
   Copy the URL, paste to the user. They open it in Happ.

### Disable / re-enable a user

- Users page → Edit → uncheck `Active`. They can't log in.
- To revoke VPN access too: Subscriptions page → Edit → status `disabled`.
  Their UUID is removed from xray's client list within seconds of save.

### Rotate an exit node

If a node's IP is burned:

1. Nodes page → click the row → **Delete**. The Panel rewrites xray
   config without that node, restarts xray. Existing client connections
   for that country drop; clients auto-reconnect to whatever's left and
   on the next subscription pull see the new list.
2. Provision a new node in the same country with the SSH form. It gets a
   fresh inbound port and a new `short_id`.
3. Subscription URLs automatically include the new node next time Happ
   pulls — no user action needed if they have auto-update enabled.

### Top up someone's traffic / extend their plan

Subscriptions page → Edit row → either:

- Bump `traffic_limit_gb` (and tick **Reset traffic** if they're already
  over).
- Tick **Extend by plan** to add `plan.duration_days` to `expires_at`.

Save. The `enforce_limits_task` will re-evaluate `expired/over_limit`
within 30 s.

## Troubleshooting playbook

The order below matches the layers traffic crosses.

### Symptom: "Happ won't connect, says server closed connection"

Means TCP reached the panel host but the relay either isn't listening
on the right port or rejected the Reality handshake.

```bash
# 1. Is xray actually listening on the inbound port?
ss -ltnp | grep xray
# Expect: *:10001 (or whatever the node's panel_inbound_port is) + 127.0.0.1:10085

# 2. Are there errors at startup?
journalctl -u xray -n 30 --no-pager
# Look for: "failed to start inbound", "REALITY", "privateKey"

# 3. Does the live config have the user's UUID?
docker compose exec postgres psql -U panel -d panel -c \
  "SELECT id, xray_uuid FROM subscriptions WHERE status='active';"
jq '.inbounds[] | select(.protocol=="vless") | .settings.clients[].id' \
  /usr/local/etc/xray/config.json
# UUIDs must match.
```

If config-on-disk has the UUID but `ss` doesn't show the port listening,
the watcher couldn't reload xray. See "watcher silently refusing to
restart" below.

### Symptom: "watcher silently refusing to restart"

```bash
journalctl -t xray-watcher --since "30 min ago"
```

If you see `config invalid or empty; skipping restart`, the validator
command (`xray run -test -config <file>`) returned non-zero. Reproduce
manually:

```bash
/usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json
```

If you see `xray test: unknown command` — your watcher script is from
before the v26.3 CLI rename. Patch:

```bash
sed -i 's|/usr/local/bin/xray test -config|/usr/local/bin/xray run -test -config|' \
    /usr/local/bin/xray-watcher.sh
systemctl restart xray-watcher
```

### Symptom: "traffic stats stuck at 0, no green online dot"

```bash
# Is the worker alive?
docker compose ps arq-worker
docker compose logs --tail=30 arq-worker
```

Common error in the logs:

- `failed to dial host.docker.internal:10085` — xray's API is bound to
  127.0.0.1 instead of 0.0.0.0. The fix is in `relay_config.py`
  (`API_INBOUND.listen = "0.0.0.0"`); if the live config still has
  127.0.0.1, regenerate via admin UI → Nodes → any → Save.
- `Refusing to start with insecure config (PANEL_ENV=production)` — the
  worker doesn't see one of the required prod env vars. Check
  `docker-compose.yml` — `arq-worker.environment` must include
  `JWT_SECRET`, `ADMIN_*`, `PANEL_HOST`, `PANEL_REALITY_*`. Latest code
  uses a YAML anchor; older versions had them omitted.
- `0.02s ← cron:collect_traffic_task ●` and DB shows 0 bytes despite
  traffic being generated → the worker is calling xray but parsing
  nothing. Likely the regex in `traffic_collector.py` expects text
  output but xray returned JSON. Fixed in commit `5d7633e` — pull and
  rebuild the worker.

Direct test from inside the worker:

```bash
docker compose exec arq-worker sh -c \
  "xray api statsquery --server host.docker.internal:10085 --pattern 'user>>>' 2>&1"
```

You should see JSON with `user>>>{email}>>>traffic>>>uplink/downlink`
entries. Empty `{}` is fine if nothing is connected right now.

### Symptom: "subscription URL works in Happ but no internet through tunnel"

The relay → exit leg is broken. Check from the Panel:

```bash
NODE_IP=185.74.45.200  # whatever your node is

# Can we reach :443 at all?
timeout 5 bash -c "</dev/tcp/$NODE_IP/443" && echo "TCP OK" || echo "TCP FAIL"

# Does the exit answer TLS?
echo | timeout 5 openssl s_client -connect "$NODE_IP:443" 2>/dev/null \
  | openssl x509 -noout -subject

# Does the s2s password in DB match what xray on the exit has?
docker compose exec -T postgres psql -U panel -d panel -c \
  "SELECT host, left(s2s_password, 8)||'...' FROM nodes;"
ssh root@$NODE_IP "jq '.inbounds[0].settings.clients[0].password' \
  /usr/local/etc/xray/config.json"
```

TCP fail → UFW on exit isn't allowing Panel IP. Re-check the
provisioner log — `ufw allow from <PANEL_IP> to any port 443 proto tcp`
must have succeeded.

### Symptom: "Happ rejects the subscription URL: HTTP scheme forbidden"

Happ requires `https://`. The panel emits `http://` when
`request.base_url` is used and Caddy isn't terminating TLS, or when
`PANEL_ENV` isn't `production`. Fix:

```bash
docker exec panel-panel-api-1 env | grep ^PANEL_ENV
# Expect: PANEL_ENV=production
```

If empty, your `docker-compose.yml` isn't passing `PANEL_ENV` to
`panel-api`. Update to latest and `up -d --build panel-api`.

## Known limits

- **Single point of failure**: the Panel. Snapshot it nightly; have a
  redeploy script ready. Active-active HA is not implemented (the
  xray-in-memory state would need to be replicated, which is a
  non-trivial design).
- **Throughput is gated by Panel ↔ Exit pipe** (TCP-over-TCP). At ~50ms
  RTT and ~1% packet loss between the Panel and an exit, a single
  client speedtest will see 20–60 Mbps even on 1 Gbit uplinks. See
  performance tuning below.
- **No device binding** on subscriptions. A user can share their URL
  with as many devices as they like; only their traffic + time quota
  limits abuse.
- **No payment integration**. The `payments` model is intentionally
  not present until a provider is wired up.
- **xray version drift**: the panel image bundles `xray` v1.8.23 for
  the CLI calls (statsquery), the host runs whatever `install_panel.sh`
  installed (latest). They speak the same gRPC, but if you bump the
  host xray to a major version that drops the v1 CLI flags, the
  collector will start failing — pin the host version in
  `install_panel.sh` if that worries you.

## Performance tuning

Default Linux on a fresh VPS is **not** tuned for a high-RTT lossy VPN
relay. Easy wins:

### 1. BBR + larger TCP buffers (both Panel and Exit)

```bash
cat >/etc/sysctl.d/99-vpn-perf.conf <<'EOF'
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
net.core.rmem_max=67108864
net.core.wmem_max=67108864
net.ipv4.tcp_rmem=4096 87380 67108864
net.ipv4.tcp_wmem=4096 65536 67108864
net.ipv4.tcp_mtu_probing=1
net.ipv4.tcp_notsent_lowat=131072
net.ipv4.tcp_fastopen=3
EOF
sysctl --system
```

Expect 2–5× improvement over default cubic on a high-RTT (>30 ms)
link.

### 2. Measure the actual Panel ↔ Exit ceiling

```bash
# On the exit (allow port temporarily)
ufw allow 5201/tcp
iperf3 -s

# On the Panel
iperf3 -c <exit-ip> -t 20 -P 4

# Cleanup
# Exit: pkill iperf3; ufw delete allow 5201/tcp
```

The `[SUM] Bitrate` line is the **physical ceiling** for any VPN traffic
between those two boxes. If it's < 100 Mbps, no protocol tuning will
make VPN faster than that — the link is the problem (rent a different
hoster, try a closer country, or move the exit).

### 3. If the link is fine but VPN is slow → switch transport

Trojan-over-TLS between Panel and Exit is TCP-in-TCP. On a lossy link
this stalls badly. Realistic alternatives:

- **WireGuard between Panel and Exit** — UDP, kernel crypto. Best
  performance. Requires rewriting `bootstrap_node.sh` to install
  `wg-quick`, and the `_outbound_for_node` template in `relay_config.py`
  to emit a `wireguard` outbound instead of `trojan`. Not implemented
  in main yet.
- **mKCP (XRay-native UDP transport)** between Panel and Exit. Smaller
  change but more overhead than WG.

### 4. CPU pressure on the relay

```bash
top -b -n 1 -p $(pidof xray)
```

If `%CPU` hits 90+ on a single core during peak, the relay is
CPU-bound. xray is single-core-bound per connection on Reality; multi-
client throughput is fine on multiple cores, but a single fat client
download can't be parallelized. Scaling answers:

- Move to a bigger-CPU VPS (8+ cores).
- Run xray with `GOMAXPROCS=N` matching the box (default = all cores;
  usually already optimal).
- Accept it — single-stream is rarely the user's complaint.

## Monitoring (minimal)

Today's monitoring story is "watch the admin dashboard + tail logs".
A reasonable next step:

- **Healthcheck pinger** (UptimeRobot / Healthchecks.io) pointed at
  `https://<panel>/healthz`. Pages you if it stops returning 200.
- **Disk + RAM** alarms via the hoster's dashboard.
- **Postgres bloat** — schedule a `VACUUM ANALYZE` weekly. Traffic logs
  grow ~one row per minute per active sub; at thousands of subs you'll
  want a retention policy (out of scope for now).

For more, Prometheus + Grafana would be the next step — xray-core can
expose stats over Prometheus via a third-party exporter. Not wired in.
