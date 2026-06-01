# Deployment

## Prerequisites

- A **fresh Ubuntu 22.04 / 24.04** root shell (Debian 12 also works).
  Re-using a server that already has docker / nginx / fail2ban / etc.
  half-configured is asking for trouble — start clean.
- A **public IPv4** (IPv6 too, but the panel uses v4-only logic in places).
- A **DNS A record** pointing your chosen hostname (e.g. `vpn.example.com`)
  at that IP. Caddy will not get a certificate without it.
- **Symmetric ≥1 Gbit** uplink with a generous monthly cap — the Panel is
  the bandwidth bottleneck for *all* your clients.

That's it. No external Postgres, no Redis cluster, no Vault — secrets
land in `/root/.panel-secrets.env` (chmod 600) on first run.

## The one command

```bash
PANEL_HOST=vpn.example.com \
bash <(curl -fsSL https://raw.githubusercontent.com/Arcadnick/Internet-accelerator/main/deploy.sh)
```

That's the entire happy path. The script is idempotent — running it again
after a code update will pull `main`, rebuild, and roll the stack without
regenerating secrets.

## What `deploy.sh` actually does

In order, the script:

1. **Disables preinstalled SSH bouncers** (fail2ban, sshguard, crowdsec).
   Many VPS images ship with one of these, and they happily ban your laptop
   mid-deploy when you `ssh` in a few times. We turn them off; if you want
   ban-on-failure later, layer something like `endlessh` instead.
2. **Installs Docker + the compose plugin** from Docker's apt repository
   (not Snap, not Ubuntu's lagging deb).
3. **Configures a Docker registry mirror** (`mirror.gcr.io`) in
   `/etc/docker/daemon.json`. Shared cloud IPs run into Docker Hub's
   anonymous rate-limit constantly; the mirror routes pulls through
   Google's anonymous-friendly cache.
4. **Opens 22 / 80 / 443 / 10001-10999 in UFW**. Note: **10085 is NOT
   opened** — that's the XRay gRPC API, reachable only from the Panel's
   own Docker bridge.
5. **Clones the repo** to `/opt/accelerator` (or updates if it's already
   there). Code on subsequent runs comes from `origin/main` via
   `git reset --hard`.
6. **Runs `panel/scripts/install_panel.sh`** if `xray` is missing on the
   host. That installer:
   - Downloads the latest `xray-core` release for your arch
     (x86_64 or arm64).
   - Writes a minimal `/usr/local/etc/xray/config.json` (api inbound only).
   - Generates a Reality keypair via `xray x25519` and prints it.
   - Installs `xray.service` (systemd unit running the relay).
   - Installs `xray-watcher.service` — an inotify loop that, whenever
     `config.json` is rewritten by the panel container, validates the new
     config with `xray run -test -config <path>` and `systemctl restart xray`s
     atomically.
7. **Generates secrets — once — into `/root/.panel-secrets.env`**:
   - `JWT_SECRET` (256-bit hex)
   - `ADMIN_PASSWORD` (20 chars alnum)
   - `POSTGRES_PASSWORD` (256-bit hex)
   - `PANEL_REALITY_PRIVATE_KEY` and `PANEL_REALITY_PUBLIC_KEY` (extracted
     from xray's config if it was there already, otherwise freshly generated)

   On subsequent runs the script re-uses these, so re-deploying the same
   server doesn't invalidate existing subscription URLs.
8. **Writes `panel/.env`** by interpolating the secrets and your
   `PANEL_HOST` / `ADMIN_EMAIL` overrides.
9. **`docker compose up -d --build`** with both compose files
   (`docker-compose.yml` + `docker-compose.prod.yml`). The prod overlay
   adds Caddy (auto-TLS via Let's Encrypt) and bind-mounts the host's
   `/usr/local/etc/xray` into the panel container so config writes hit
   the host's xray.
10. **Waits up to 120 s** for `panel-api`'s `/healthz` to return `ok`,
    then prints the admin URL + password.

## After deploy — first browser visit

1. Open `https://<your-panel>/admin`. Caddy will spend a few seconds
   getting the cert — first hit may take 10–15 s.
2. Sign in with the admin email and the password printed by the script.
   The password is also in `/root/.panel-secrets.env` if you missed it.
3. **Change the admin password** immediately — Users → admin@... → Edit.

## Adding the first exit node

Two modes, both in **Admin → Nodes**:

### Mode A: SSH auto-provisioning (recommended for fresh VPS)

1. Get a fresh Ubuntu 22.04/24.04 VPS in your target country. Note its
   public IP and root password (or SSH key).
2. Click **Add node via SSH**.
3. Fill: host, root user (`root`), password OR private key paste, country
   code (`DE`, `NL`, …), label (`Frankfurt`).
4. Submit. The page streams live logs over Server-Sent Events as the panel:
   - SSHes in, uploads `bootstrap_node.sh`, runs it.
   - Bootstrap installs xray, writes a self-signed cert, generates a
     random Trojan password, configures `systemd` + UFW (deny all,
     allow ssh, allow 443 from Panel-IP only).
   - Bootstrap prints a final JSON line with `s2s_password` and `host`.
5. Panel parses the JSON, inserts the row in `nodes`, allocates an
   inbound port (lowest free in 10001–10999), generates a Reality
   short-id, regenerates the Panel's xray config, and the watcher
   restarts xray. Live.

Root creds are held in the panel-api process memory **for the duration
of the SSH session only**. They are never persisted to disk or DB. After
bootstrap the only way back into the exit is via SSH key (if you supplied
one via the `admin_ssh_key` field) or the hoster's console.

### Mode B: Manual (you already have an xray box you want to register)

In **Admin → Nodes → Add node manually**, fill in host, s2s password,
SNI, etc. The panel only writes the DB row + regenerates xray; it
doesn't touch the exit. Use this when you want to test against an existing
xray-Trojan box, or for nodes you provisioned with a different tool.

## Production checklist

- [ ] Domain DNS A record points at the Panel IP, propagated.
- [ ] You can `curl https://<host>/healthz` from your laptop and get `ok`.
- [ ] Admin password changed from the auto-generated default.
- [ ] At least one Plan exists and is `active`.
- [ ] At least one Node exists and shows `status=active`.
- [ ] `/root/.panel-secrets.env` is **backed up** somewhere off-box.
  If you lose it and the postgres volume, you lose all auth + Reality
  keys, which invalidates every existing subscription URL.
- [ ] `docker compose ps` shows all 5 containers healthy.
- [ ] `journalctl -u xray -n 20` is clean (no `failed` / `error`).
- [ ] `journalctl -u xray-watcher -n 5` shows `config changed, xray restarted`
  after the first node was added.

## Re-deploys, code updates

When you push code changes to `main` and want them on the server:

```bash
cd /opt/accelerator
git pull
cd panel
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

If you changed `relay_config.py` and want the live xray config to reflect
the new template, you need to trigger a rebuild — easiest is to go to
the admin UI, open any node, hit **Save** (no edits needed). That calls
`rebuild_and_apply()` which writes the new config and the watcher restarts
xray.

If `arq-worker` only — `--build arq-worker` is enough.

## Disaster recovery (single-box)

The two pieces of state are:

1. **Postgres volume** (`pg_data`). All users, plans, subs, traffic logs.
2. **`/root/.panel-secrets.env`**. JWT secret, admin password, Reality
   keys, Postgres password.

Lose (2) and Postgres still has the data — but you can't *decrypt* or
*regenerate* the existing Reality config, which means every subscription
URL out in the wild becomes useless (clients hit Reality handshake errors).

Recommended setup:

```bash
# nightly to your laptop / S3:
ssh root@panel "docker exec panel-postgres-1 \
   pg_dump -U panel -d panel" | gzip > panel-$(date +%F).sql.gz
scp root@panel:/root/.panel-secrets.env panel-secrets-$(date +%F).env
```

To restore on a new box with the same hostname:

```bash
# 1. run deploy.sh — it'll generate fresh secrets, IGNORE them
PANEL_HOST=vpn.example.com bash <(curl -fsSL .../deploy.sh)

# 2. overwrite the secrets file with your backup
cp panel-secrets-2026-05-28.env /root/.panel-secrets.env

# 3. re-run deploy.sh — it'll see existing secrets and reuse them
PANEL_HOST=vpn.example.com bash <(curl -fsSL .../deploy.sh)

# 4. restore the DB
docker exec -i panel-postgres-1 psql -U panel -d panel < panel-2026-05-28.sql

# 5. trigger a config rebuild (admin UI → any node → Save)
```

## Scaling notes (when you outgrow one box)

The Panel is intentionally a single box today. When it isn't enough:

- **More clients, no other geographic concern**: vertical scale —
  bigger CPU/uplink VPS. xray relay is largely CPU-bound on single
  cores; 4 cores ≈ 4× concurrent throughput.
- **Geographic spread of clients**: put a CDN-style "regional Panel" close
  to clusters of users (e.g. one in EU, one in Asia). Each panel is
  independent, has its own DB and its own pool of exits. Users get
  subscription URLs from "their" panel. This is a forklift, not a
  configuration change.
- **HA on a single region**: pair two boxes behind an L4 load balancer
  with shared Postgres and Redis. The xray state on both must stay in
  sync — `rebuild_and_apply()` would need to run on both panels on
  every change. Not implemented; design TBD.

## Useful commands on the box

```bash
# Status
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps
systemctl status xray xray-watcher
ss -ltnp | grep -E '10001|10085|443|8000'

# Logs
docker compose logs -f panel-api
docker compose logs -f arq-worker
journalctl -u xray -f
journalctl -u xray-watcher -n 20 --no-pager
docker compose logs caddy --tail=50

# Restart selectively
docker compose restart panel-api
systemctl restart xray              # picks up /usr/local/etc/xray/config.json
systemctl restart xray-watcher

# DB shell
docker compose exec postgres psql -U panel -d panel

# Force-rebuild xray config (from inside the panel container)
docker compose exec panel-api python -c "
import asyncio
from app.db import AsyncSessionLocal
from app.services.xray_local import rebuild_and_apply
async def main():
    async with AsyncSessionLocal() as db:
        await rebuild_and_apply(db)
asyncio.run(main())
"
```

See [operations.md](operations.md) for day-2 debugging.
