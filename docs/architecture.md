# Architecture

## TL;DR

One main server (the **Panel**) is both the admin/billing panel and the
XRay-core **relay**. Exit nodes in other countries are dumb forwarders that
only accept traffic from the Panel's IP. Clients connect to **the Panel
only**, pick a country in their Happ app, and the Panel proxies them to
the chosen exit. Users, accounting, and limits live entirely on the Panel —
exits have no clientele.

## Why this shape and not a federation

The straightforward design would be: each country = its own node, each node
runs its own xray + has its own subscription, the admin panel pushes users
to all of them. We **rejected** that for three concrete reasons:

1. **One IP for the client.** Everywhere the client opens a socket goes to
   `panel.example.com`. In networks that classify-by-IP-block (some Russian
   ISPs, university captive portals, hotel WiFi), being able to swap the
   exit *without* changing the IP the client talks to is significant.

2. **One stats database.** All traffic flows through the Panel's xray;
   per-user uplink/downlink is captured on a single counter. We don't have
   to gather stats from N nodes, merge them, deduplicate, handle nodes
   going briefly offline, etc.

3. **Exit nodes are throwaway.** They contain no user list, no panel,
   no billing logic. If one gets seized, fingerprinted, or its IP burned,
   we just delete it from the DB and provision a new one. Subscription
   URLs in users' Happ apps re-pull and pick up the new country list
   transparently.

The cost is explicit and accepted:

- **Panel is a bandwidth bottleneck.** All client traffic passes through it.
  Plan on a ≥1 Gbit symmetric uplink with generous monthly cap.
- **Extra hop adds latency.** Typical 5–30 ms more than direct-to-exit.
- **Panel = SPOF.** Losing it kills the service. The mitigation is fast
  redeploy from snapshot, not active-active HA (which is a separate epic).

If you need any of those constraints to be wrong, this isn't the right
shape and you'd be better off with a federation.

## Trust boundaries

```
                        ┌──────── trust = NONE ───────┐
                        │ client device / Happ        │
                        └──────────┬──────────────────┘
                                   │
       authenticated by UUID       │
       in VLESS+Reality            │
                                   ▼
                        ┌──────── trust = HIGH ───────┐
                        │ Panel (control + data)      │
                        │  • holds user DB, plans,    │
                        │    billing, Reality privkey │
                        │  • runs admin/auth          │
                        │  • routes per-country       │
                        └──────────┬──────────────────┘
                                   │
       authenticated by Trojan     │
       password + UFW src-ip       │
                                   ▼
                        ┌──────── trust = MEDIUM ─────┐
                        │ Exit node                   │
                        │  • dumb forwarder           │
                        │  • knows ONE password       │
                        │  • doesn't see clients      │
                        └─────────────────────────────┘
```

- **Client → Panel** is mutually unauthenticated *as the TLS layer pretends
  to be a real website* (Reality). The client authenticates with its
  per-subscription UUID inside the VLESS handshake.
- **Panel → Exit** has two layers of access control:
  1. UFW on the exit denies inbound on :443 from anyone except `PANEL_IP`.
  2. The Trojan password — a 32-byte random hex generated per node.
- **Panel ↔ XRay (local)** is loopback gRPC on :10085. The API is bound to
  `0.0.0.0` so the panel-api Docker container can reach it via
  `host.docker.internal`; external access is blocked by UFW (10085 is not
  in the allowlist).

A compromised exit can only see traffic destined for and from that one
country. It can sniff cleartext payloads (HTTPS to user destinations
already terminates *after* the exit, so the exit sees plaintext from the
TLS hand-off perspective — same as any VPN exit). It cannot enumerate
users or pivot to other countries.

A compromised Panel = full compromise. Treat its disk/snapshots as
sensitive.

## Data plane

```
client ─VLESS+Reality─▶ Panel:10001  ── routing rule ──▶  out-de
                       (inbound tag                       (outbound to
                        = "in-de-XXXX")                    de-node:443
                                                          via Trojan/TLS)
                                                                │
                                                                ▼
                                                          de-node:443
                                                                │
                                                          freedom outbound
                                                                ▼
                                                          internet (DE IP)
```

For each node `N` in the database the panel generates **two** entries in
the relay's XRay config:

- An **inbound** on port `panel_inbound_port` (allocated from
  `10001-10999`) with protocol `vless`, transport `tcp`, security `reality`,
  and one `client` entry per active subscription (`id = sub.xray_uuid`,
  `email = sub.xray_email`).
- An **outbound** with protocol `trojan` pointing at `node.host:443`, using
  the `s2s_password` recorded at provisioning time.
- A **routing rule** `inboundTag=[N.inbound_tag] → outboundTag=N.outbound_tag`.

Per-user stats are enabled via
`policy.levels.0.statsUserUplink/statsUserDownlink`, so xray records
`user>>>{email}>>>traffic>>>uplink` and `…>>>downlink` per active
subscription. The `arq-worker` polls these counters once a minute and
persists the deltas into `traffic_log` + `subscriptions.traffic_used_bytes`.

## Control plane

The Panel is the source of truth for *everything*. Exit nodes hold no state
that isn't reproducible from the DB.

- **Panel ↔ Panel-XRay**: localhost gRPC on `10085`. The panel writes the
  whole config to `/usr/local/etc/xray/config.json` and the
  `xray-watcher.service` (host-side inotify watcher) validates and
  restarts. New users could also be added via gRPC `AddUser` without
  a restart — currently we rewrite + restart because reasoning about
  partial state divergence between xray-in-memory and DB-on-disk is
  more bug-prone than just rebuilding.
- **Panel ↔ Exit node — bootstrap**: AsyncSSH session run by
  `services/provisioner.py`. It uploads `scripts/bootstrap_node.sh`,
  executes it (root creds held in process memory only, never written to
  disk or DB), and parses the final JSON line for the s2s credentials.
- **Panel ↔ Exit node — steady state**: none. Once provisioned, the exit
  is independent. We don't health-check from the Panel today; if an exit
  goes down, its inbound on the Panel stays open but outbound dials fail,
  Happ surfaces a connection error to the user, and the admin notices via
  the dashboard.

## Subscription URL format

`GET /sub/<token>` returns a base64-encoded list of `vless://` URIs (one per
node), sing-box-flavored. Happ accepts both base64 and JSON; we serve
base64 because it's the smallest and most universal.

Each URI shares the same `uuid` (the subscription's `xray_uuid`) and differs
only in port + `sid` (Reality short ID) + name:

```
vless://<uuid>@vpn.example.com:10001
  ?type=tcp
  &security=reality
  &flow=xtls-rprx-vision
  &pbk=<panel_reality_pubkey>
  &sni=www.microsoft.com
  &sid=<node.reality_short_id>
  &fp=chrome
  &encryption=none
#NL%20Netherlands
```

The Reality `serverName` is hardcoded to `www.microsoft.com` — picking
something Cloudflare-fronted or otherwise "looks like real traffic" matters
in censored networks. Override via `PANEL_REALITY_SERVER_NAME` if needed.

## Database schema (one-paragraph summary)

- `users(id, email, password_hash, role, is_active, …)` — admins + clients
- `plans(id, name, traffic_bytes, duration_days, price, is_active)`
- `subscriptions(id, user_id, plan_id, xray_uuid, xray_email, sub_token,
  status, traffic_used_bytes, traffic_limit_bytes, expires_at, last_seen_at, …)`
- `nodes(id, country_code, label, host, ssh_port, status, s2s_password,
  s2s_sni, s2s_allow_insecure, panel_inbound_tag, panel_outbound_tag,
  panel_inbound_port, reality_short_id, last_seen_at, …)`
- `traffic_log(id, subscription_id, bytes_up, bytes_down, collected_at)` —
  append-only, used for charts; the running total lives on `subscriptions`
- `node_events(id, node_id, kind, message, created_at)` — provisioning audit

Migrations live under `panel/alembic/versions/` and run automatically on
container start (`alembic upgrade head` in panel-api's command).

## Background jobs (ARQ)

Two cron jobs in `panel/app/workers/tasks.py`:

- **`collect_traffic_task`** — every 60 s. Runs
  `xray api statsquery --pattern 'user>>>' --reset` against the Panel's
  gRPC API, parses the JSON output, and adds deltas to
  `subscriptions.traffic_used_bytes`. Setting `last_seen_at = now()` on any
  sub that had non-zero traffic this cycle is what drives the "online"
  green dot in the admin UI.
- **`enforce_limits_task`** — every 30 s. Marks subs `expired` if
  `expires_at < now()`, `over_limit` if `traffic_used >= traffic_limit`,
  and triggers a relay config rebuild so the over-quota client is removed
  from xray's in-memory client list within a minute.

## Threat model — what this protects against, and what it doesn't

Protects against (in scope):

- **Passive censor that blocks "known VPN hosts" by IP**: Reality makes
  the panel look like a real TLS connection to `www.microsoft.com` (or
  whatever you set as `PANEL_REALITY_SERVER_NAME`). Active probing returns
  the real Microsoft TLS handshake because Reality forwards unauthenticated
  connections through.
- **Per-user accounting / quotas / time limits**: enforced server-side,
  unforgeable by clients.
- **One leaky exit doesn't expose all users**: exits hold no client list.

Does **not** protect against (out of scope):

- **A nation-state targeting your specific Panel host**: if your IP is
  burned, the relay model concentrates risk. Plan rotation / mirrors.
- **Compromise of the Panel server**: full compromise. Treat
  `/root/.panel-secrets.env` and the Postgres volume as crown jewels.
- **Malicious clients sharing UUIDs**: a single subscription URL handed to
  multiple devices works. Traffic + time limits put a cap on the abuse;
  there's no device-binding today.
- **Traffic correlation by an observer with taps on both sides**: same
  as any single-hop VPN. Tor or a multi-hop mesh would be a different
  architecture, not in scope here.

## File map of the moving parts

| Concern                       | File                                                                                      |
|-------------------------------|-------------------------------------------------------------------------------------------|
| Generate XRay JSON from DB    | [panel/app/services/relay_config.py](../panel/app/services/relay_config.py)               |
| Write config + trigger reload | [panel/app/services/xray_local.py](../panel/app/services/xray_local.py)                   |
| Pick free port + short-id     | [panel/app/services/node_allocator.py](../panel/app/services/node_allocator.py)           |
| Build subscription URLs       | [panel/app/services/subscription_builder.py](../panel/app/services/subscription_builder.py) |
| New subscription side effects | [panel/app/services/subscription_factory.py](../panel/app/services/subscription_factory.py) |
| SSH bootstrap of exits        | [panel/app/services/provisioner.py](../panel/app/services/provisioner.py)                 |
| Collect per-user traffic      | [panel/app/services/traffic_collector.py](../panel/app/services/traffic_collector.py)     |
| Expire / over-limit logic     | [panel/app/services/billing.py](../panel/app/services/billing.py)                         |
| Host XRay + watcher install   | [panel/scripts/install_panel.sh](../panel/scripts/install_panel.sh)                       |
| Exit-node setup               | [panel/scripts/bootstrap_node.sh](../panel/scripts/bootstrap_node.sh)                     |
