# Development

## Local stack

The compose stack runs fine on a laptop:

```bash
cd panel
cp .env.example .env  # if you have one; otherwise hand-write — see below
docker compose up -d --build
docker compose logs -f panel-api
```

`panel-api` is then on <http://localhost:8000>. The admin UI is at
`/admin`, Swagger at `/docs`.

Minimum `.env` to make the stack come up in dev:

```env
PANEL_ENV=dev

POSTGRES_USER=panel
POSTGRES_PASSWORD=panel
POSTGRES_DB=panel

JWT_SECRET=dev-secret-change-me
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=admin
PANEL_HOST=localhost

# Leave blank for dev — xray on the host won't be reachable from
# the container anyway; the relay parts are no-ops without it.
PANEL_REALITY_PRIVATE_KEY=
PANEL_REALITY_PUBLIC_KEY=
PANEL_XRAY_API_ADDR=host.docker.internal:10085
```

`PANEL_ENV=dev` keeps the prod-hygiene validator quiet. With `dev`:

- The session cookie is **not** marked `Secure` (so `http://localhost`
  works).
- `public_base_url()` uses `request.base_url` (i.e. whatever Host header
  came in) instead of forcing `https://PANEL_HOST`.
- The validator skips the "no default secrets" check.

## Migrations

Alembic runs automatically on container start
(`alembic upgrade head` in `panel-api`'s command). To author a new
migration:

```bash
# After editing app/models/*.py
docker compose exec panel-api alembic revision --autogenerate -m "add foo to bar"
# Inspect the generated file under panel/alembic/versions/ — autogenerate
# is fuzzy; check the upgrade()/downgrade() match what you actually want.
docker compose exec panel-api alembic upgrade head
```

Migrations are linear, no branching. Numeric prefix in the filename
(`0001_…`, `0002_…`) keeps the order stable.

## Tests

Test infra is wired up (`pytest-asyncio` in deps, `tool.pytest`
configured for `asyncio_mode = "auto"`) but **no tests are in the repo
yet**. The pragmatic order to add them:

1. `services/relay_config.py` — pure function `(nodes, subs, settings) → dict`.
   Trivial to test with hand-built model instances.
2. `services/subscription_builder.py` — same. Pure transform.
3. `services/traffic_collector._query_user_stats` — mock the
   `xray api statsquery` subprocess output.
4. API endpoints — `httpx.AsyncClient(app=app)` against an in-memory
   SQLite or a disposable Postgres.

To run: `pytest panel/` from the repo root, or
`docker compose exec panel-api pytest /app` inside the container.

## Code conventions

Linter is `ruff` with `E, F, I, B, UP, N, SIM`:

- `E/F`: PEP 8 + pyflakes basics.
- `I`: import order. Use `ruff check --fix` or your IDE on save.
- `B`: bugbear (catches common footguns).
- `UP`: prefer modern Python idioms.
- `N`: naming.
- `SIM`: simplifications.

Line length is 100. Type hints are expected on function signatures
(this is a Python 3.12 codebase, use `X | None`, not `Optional[X]`).

Mypy is in dev deps but not enforced in CI yet. If you add it, configure
`strict_optional` and `disallow_untyped_defs`.

Run all of it:

```bash
cd panel
docker compose exec panel-api ruff check app
# or locally if you have a venv
ruff check panel/app
```

## Repo conventions

- **Server-rendered HTML over SPA**. The admin UI is Jinja2 + HTMX +
  Alpine.js. Reasoning: tiny attack surface, no build pipeline, every
  state change is a server-issued page so refresh-after-action works
  without invalidating cache.
- **Cookie session for browser, JWT bearer for API**. The same `session`
  cookie carries a JWT inside it (set on POST /login); the API path
  reads it via `OAuth2PasswordBearer` if you submit `Authorization`.
- **Idempotent admin actions**. Every CRUD that touches xray config
  calls `services.xray_local.rebuild_and_apply()` *after* the DB commit.
  This keeps DB-on-disk and xray-in-memory in sync without partial
  state divergence.
- **Atomic config writes** (`os.replace` of a tmp file). The
  inotify watcher only fires on `MOVED_TO` / `close_write`, so partial
  writes can't trigger a bad reload.
- **No comments for what code does**, only for *why* something
  non-obvious is the way it is (subtle invariants, workarounds for
  specific upstream bugs, threat-model rationale).

## Things to know before changing something

- **`relay_config.py`** is the heart of the system. Any change to its
  output must keep the `inbound_tag` / `outbound_tag` / `port` /
  `short_id` schemas backwards compatible — those values are baked into
  existing subscription URLs in users' Happ apps. If you must change
  them, also add a Reality keypair rotation and accept that users will
  need to re-pull subscriptions.

- **`xray_local.rebuild_and_apply()`** rewrites the *entire* xray config
  every time. We don't use xray's gRPC `AddUser/RemoveUser` because the
  reasoning about "did the DB and xray drift?" gets exponentially harder
  the more state you incrementally mutate. Restarts are cheap (xray
  starts in <1s) and clients reconnect transparently.

- **`xray-watcher.sh`** runs on the **host**, not in the container. It's
  installed by `install_panel.sh`. Bugs in it have killed the relay
  silently before; check `journalctl -t xray-watcher` first if "I
  changed something and xray didn't pick it up".

- **`OAuth2PasswordBearer(tokenUrl="/api/auth/login")`**. The Swagger
  "Authorize" button POSTs `username/password` as form-data to that
  tokenUrl. The login endpoint currently consumes JSON, so the Swagger
  Authorize flow won't work as-is; for API testing you need to call
  `/api/auth/login` via "Try it out" with JSON, then there's no clean
  way to apply the token within Swagger without changing the endpoint.

## How to add a new admin page

1. Add (if needed) a SQLAlchemy model in `app/models/`.
2. Add the alembic migration (see Migrations above).
3. Add a Pydantic schema in `app/schemas/` for the JSON API path.
4. Add a JSON API router in `app/api/admin_<thing>.py` —
   `GET / POST / PATCH / DELETE` against the model.
5. Add an HTML router in `app/web_admin.py` — GET that renders the
   template, POST that consumes a form.
6. Add the Jinja template in `app/templates/admin/<thing>.html` and
   (if it's a list page) an Edit page at `app/templates/admin/<thing>_edit.html`.
7. Add translation keys to `app/i18n.py` (both `en` and `ru`).
8. Register the router in `app/main.py`.
9. Add a nav link in `app/templates/admin/base.html` if it deserves one.

The existing **Nodes** flow is the closest reference; **Plans** is the
simplest.

## How to add a new background job

1. Write the async function in `app/services/` (or `app/workers/`).
2. Add it to `WorkerSettings.cron_jobs` in `app/workers/tasks.py`.
3. `docker compose up -d --build arq-worker` — the worker picks it up on
   the next start.

The two existing jobs (`collect_traffic_task`, `enforce_limits_task`)
are the templates.

## How to test the relay end-to-end locally

You can't really — Reality needs a public IP and a real TLS handshake
to a "real" destination. The realistic dev loop is:

1. Run the **panel-api + postgres + redis** locally.
2. Deploy a **separate** test VPS (cheap, hourly) with the full
   `deploy.sh`.
3. SSH-tunnel local panel-api to the VPS's xray if you really need to
   integrate the two; otherwise treat the relay as a black box.

The unit-testable parts (config generation, subscription URL builder,
billing logic) cover most of the change surface without needing a
real xray.
