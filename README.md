# Internet-accelerator

> Self-hosted VPN service with a central relay and per-country exit nodes.
> One subscription URL — clients pick a country in-app (Happ).
> Admin panel with billing, traffic accounting, and SSH auto-provisioning of new nodes.

[![License: PolyForm Noncommercial 1.0.0](https://img.shields.io/badge/license-PolyForm%20NC%201.0.0-blue.svg)](LICENSE)

---

## What this is

A drop-in VPN backend you can deploy on a fresh Ubuntu VPS and have running
in 10 minutes:

- **One server (Panel)** acts as both the admin panel and the XRay-core relay.
  Clients connect here over VLESS+Reality.
- **N exit nodes** in different countries run XRay-core in Trojan-only mode,
  accepting connections **only from the Panel's IP** (UFW-enforced). They have
  no users, no panel, no clientele — just dumb forwarders.
- Adding a new country = one form in the admin UI; the panel SSHes in,
  bootstraps the node, and registers it. Clients pick it up automatically
  on their next subscription pull.

Clients never see the IPs of exit nodes — only the Panel host. That makes
it harder to fingerprint as "a VPN" and lets you rotate exits without
breaking any subscription.

```
                       Happ (client)
                            │
                            │ VLESS+Reality (TLS-as-Reality)
                            ▼
       ┌───────────────────────────────────────────────────────┐
       │  PANEL (single server)                                │
       │  ┌─────────┐  ┌──────────┐  ┌────────┐                │
       │  │ FastAPI │  │ Postgres │  │ Redis  │ ← admin / API  │
       │  └─────────┘  └──────────┘  └────────┘                │
       │  ┌──────────────────────────────────────────────────┐ │
       │  │  XRay-core (relay)                               │ │
       │  │   inbound :10001 in-de  ─┐   routing             │ │
       │  │   inbound :10002 in-nl  ─┼─▶ inboundTag →        │ │
       │  │   inbound :10003 in-us  ─┘   outboundTag         │ │
       │  │                                                  │ │
       │  │   outbound out-de  ─▶ de-node:443  (Trojan/TLS)  │ │
       │  │   outbound out-nl  ─▶ nl-node:443                │ │
       │  │   outbound out-us  ─▶ us-node:443                │ │
       │  └──────────────────────────────────────────────────┘ │
       └─────────────┬─────────────┬───────────┬───────────────┘
                     │             │           │  ufw allows panel-ip only
                     ▼             ▼           ▼
                 ┌───────┐    ┌───────┐    ┌───────┐
                 │ DE-EX │    │ NL-EX │    │ US-EX │
                 └───┬───┘    └───┬───┘    └───┬───┘
                     ▼            ▼            ▼
                  internet     internet     internet
                  (DE IP)      (NL IP)      (US IP)
```

## Status

Works end-to-end. The dashboard, billing, traffic accounting, online indicator,
and SSH auto-provisioning are all wired up.

Known limits — see [docs/operations.md](docs/operations.md#known-limits) for details:

- Single-server panel (no HA yet). Snapshot + redeploy is the recovery story.
- Throughput is gated by Panel ↔ Exit TCP-over-TCP latency; tunable
  (BBR + sysctl + transport choice — see [docs/operations.md](docs/operations.md#performance-tuning)).
- No payment integration yet; the `payments` table is intentionally absent
  until a provider is wired up.

## Quick deploy (recommended path)

On a **fresh Ubuntu 22.04/24.04 root shell**, with a domain (e.g.
`vpn.example.com`) already A-pointed at the box:

```bash
PANEL_HOST=vpn.example.com \
bash <(curl -fsSL https://raw.githubusercontent.com/Arcadnick/Internet-accelerator/main/deploy.sh)
```

That single script (idempotent — safe to re-run):

1. Disables fail2ban / sshguard / crowdsec (they block SSH/deploys silently).
2. Installs Docker, the compose plugin, UFW, jq, openssl, inotify-tools.
3. Configures a Docker Hub mirror (`mirror.gcr.io`) to avoid pull rate-limits.
4. Opens 22 / 80 / 443 / 10001-10999 in UFW.
5. Clones this repo to `/opt/accelerator`.
6. Installs XRay-core on the host as `xray.service` + an inotify-driven
   `xray-watcher.service` that auto-reloads the relay on config changes.
7. Generates a Reality keypair, Admin password, JWT secret, and Postgres
   password — once, persisted in `/root/.panel-secrets.env`.
8. Brings up the `postgres + redis + panel-api + arq-worker + caddy` stack.
9. Waits for `/healthz`.
10. Prints the admin URL, email, and password.

Caddy requests a Let's Encrypt cert on first browser hit, so visit
`https://vpn.example.com/admin` once after the script finishes.

## After deploy — first run

1. **Log in** to `https://<your-panel>/admin` with the printed creds.
2. **Change the admin password** (Users → admin@... → Edit).
3. **Create a Plan** — e.g. `50 GB / 30 days`. (Or `unlimited / never expires`.)
4. **Add an exit node** — Infrastructure → Nodes → Add node via SSH.
   Give it: host, root creds (used only during bootstrap, not persisted),
   country code, label.
5. **Create a user** (Access → Users) and a subscription for them
   (Access → Subscriptions → Add).
6. **Copy the subscription URL**, open in Happ, pick the country. Done.

Detailed walkthrough: [docs/operations.md](docs/operations.md).

## Repo layout

```
.
├── deploy.sh                   # One-shot Ubuntu deployer (curl-bash entry point)
├── LICENSE                     # PolyForm Noncommercial 1.0.0
├── NOTICE                      # Copyright + third-party notices
├── docs/
│   ├── architecture.md         # Why central relay, data/control planes, threat model
│   ├── deployment.md           # Production deployment, secrets, ports, scaling notes
│   ├── operations.md           # Day-2: add nodes, debug, monitor, tune throughput
│   ├── configuration.md        # All env vars + their effect
│   └── development.md          # Local dev, migrations, tests, code layout
└── panel/
    ├── Caddyfile               # Reverse proxy + TLS for the panel
    ├── Dockerfile              # Python 3.12-slim + xray CLI (for stats queries)
    ├── docker-compose.yml      # Base stack
    ├── docker-compose.prod.yml # Production overlay (Caddy + bind mounts)
    ├── pyproject.toml          # Python deps
    ├── alembic/                # Schema migrations
    ├── scripts/
    │   ├── install_panel.sh    # Host-side XRay + systemd units + watcher
    │   └── bootstrap_node.sh   # Uploaded to each exit node by the provisioner
    └── app/
        ├── main.py             # FastAPI entrypoint + lifespan seed
        ├── config.py           # pydantic-settings + prod-hygiene guard
        ├── db.py               # async SQLAlchemy session
        ├── i18n.py             # RU/EN translations for the HTML UI
        ├── security.py         # JWT + bcrypt, OAuth2 password bearer
        ├── seed.py             # Default admin user on first boot
        ├── web.py              # Cookie-session HTML routes (login, dashboard)
        ├── web_admin.py        # Cookie-session admin HTML routes
        ├── api/                # JSON API (auth, admin CRUD, /sub/<token>)
        ├── models/             # SQLAlchemy ORM (User, Node, Plan, Subscription, …)
        ├── schemas/            # Pydantic request/response models
        ├── services/
        │   ├── relay_config.py        # generates XRay JSON from current DB state
        │   ├── xray_local.py          # atomic writer → watcher → systemctl restart
        │   ├── provisioner.py         # AsyncSSH bootstrap of exit nodes (SSE-streamed)
        │   ├── subscription_builder.py# vless:// URIs + sing-box JSON for /sub/<token>
        │   ├── subscription_factory.py# new-sub side effects (UUID, AddUser, …)
        │   ├── node_allocator.py      # picks free inbound port + short-id
        │   ├── traffic_collector.py   # `xray api statsquery` → DB
        │   └── billing.py             # expires subs at time/traffic limit
        ├── workers/tasks.py    # ARQ cron jobs (traffic + billing)
        └── templates/admin/    # Jinja2 server-rendered admin UI
```

## Documentation

| File                                            | What's in it                                                                  |
|-------------------------------------------------|-------------------------------------------------------------------------------|
| [docs/architecture.md](docs/architecture.md)   | Why a single-entry relay, trust boundaries, data + control planes, trade-offs |
| [docs/deployment.md](docs/deployment.md)       | Step-by-step deploy, what `deploy.sh` actually does, prod checklist           |
| [docs/operations.md](docs/operations.md)       | Adding nodes, debugging the watcher / stats / TLS, perf tuning, troubleshooting |
| [docs/configuration.md](docs/configuration.md) | Every env var, what it does, what happens if you leave it default             |
| [docs/development.md](docs/development.md)     | Local stack, migrations, repo conventions, where things live                  |

## License

Licensed under **[PolyForm Noncommercial 1.0.0](LICENSE)** — you can run,
modify, and redistribute the source for noncommercial use (personal,
educational, research, charity, government). Selling access to end-users,
integrating into a paid product, or any other commercial use requires a
separate commercial license from the copyright holder.

Required Notice: Copyright RubinEagle (https://github.com/Rub1nEagle)

For commercial licensing, open an issue or contact the author through GitHub.
