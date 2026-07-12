# Server Infrastructure — Agent Context

Self-hosted VPS stack. Each service lives in `compose/{name}/compose.yml`. On the server the repo is cloned to **`/opt/infra`** (FHS-canonical: each add-on package gets its own `/opt/<package>` tree, which also gives the relative bind-mount volumes a stable, backup-friendly home) — `/opt` is owned by `root`, so the directory is `chown`ed to the admin user once so everything runs without `sudo`. Layer 3 apps follow the same pattern as siblings under `/opt/<app>` or `/opt/<namespace>/<app>`. All compose files are run from the **repo root** with `--env-file .env` to load the shared env file (Docker Compose v2 loads `.env` from the compose file's directory, not cwd):

```bash
docker compose --env-file .env -f compose/watchtower/compose.yml up -d
docker compose --env-file .env -f compose/traefik/compose.yml up -d
```

## Architecture

Two-tier access model via Traefik:

- **Public** (`websecure` entrypoint, `TRAEFIK_PUBLIC_IP:443`) — Cloudflare orange/grey cloud → Traefik
- **Private** (`tailscale` entrypoint, `TRAEFIK_TAILSCALE_IP:443`) — DNS points to Tailscale IP (Cloudflare grey), only reachable within tailnet

DNS is wildcard-first: `*.${DOMAIN}` → `TRAEFIK_TAILSCALE_IP` (grey) covers every private host with no per-service record. Public hosts get an explicit A record on `TRAEFIK_PUBLIC_IP` that overrides the wildcard (a more-specific record always wins) — see **Public vs private hostnames**.

All services join the external `traefik` Docker network (created once: `docker network create traefik`).

## Conventions

**Traefik labels** — every routed service must have:

```yaml
labels:
  traefik.enable: true
  traefik.http.routers.{name}.rule: Host(`{subdomain}.${DOMAIN}`)
  traefik.http.routers.{name}.entrypoints: tailscale # or websecure
  traefik.http.routers.{name}.tls: true
  traefik.http.routers.{name}.tls.certresolver: lets-encrypt
  traefik.http.services.{name}.loadbalancer.server.port: { port }
```

**Public vs private hostnames** — a service's private UI/admin/editor lives on its **base** subdomain (`op`, `pb`, `n8n`), which the `*.${DOMAIN}` wildcard resolves to the Tailscale IP — no manual DNS. Anything that must face the public internet gets its **own dedicated subdomain** (`op-api`, `pb-api`, `hooks`) routed through `websecure`. A single hostname can't be both public and private (one A record → one IP), so never put a public router and a private router on the same host — split them across two hosts instead. The dedicated public subdomains are exactly the records added to Cloudflare by hand; `./start` prints the list for whatever was deployed (`PUBLIC_RECORDS` in [`start`](start)). When a new service exposes something publicly, add its public host to `PUBLIC_RECORDS`.

**Image versioning:**

- Critical services (Postgres, Redis, MongoDB, Traefik) → pin to major version (`postgres:18`, `mongo:8`, `traefik:v3`)
- Non-critical → `latest` is fine
- Embedded databases that ship as part of an upstream service's bundled compose (e.g. OpenPanel's `op-db`, `op-kv`, `op-ch`) → pin to the exact version the upstream tests against (`postgres:14-alpine`, `redis:7.2.5-alpine`, `clickhouse/clickhouse-server:25.10.2.65`). Their migrations target a specific version — drift can break the upstream service.
- [Watchtower](compose/watchtower/compose.yml) updates only explicitly enabled containers. Major-pinned tags receive minor/patch updates; `latest` follows the upstream tag and can cross major versions.

**Automatic updates** — Watchtower runs with `WATCHTOWER_LABEL_ENABLE=true`, so updates are opt-in. An absent `com.centurylinklabs.watchtower.enable` label means “do not update”; do not add `enable=false` as routine boilerplate. Add the following label only in server-owned compose files or deployment overlays, and only after confirming that unattended updates are safe:

```yaml
labels:
  com.centurylinklabs.watchtower.enable: true
```

Layer 3 application base compose files must remain Watchtower-agnostic; their server overlay owns the opt-in decision. Version-coupled embedded databases and one-shot jobs remain unlabeled.

**Memory limits** — every service carries a `mem_limit` (Compose v2 short form), placed right after `restart:`. It's a ceiling, not a reservation: a container consumes only what it needs, so the sum of ceilings intentionally overcommits the 8 GB box — typical concurrent usage fits with page-cache headroom, and pods that spike are contained by per-cgroup OOM (killing the offender, not a random victim) with [swap](README.md#7-hostname-timezone-swap) as the transient-spike backstop. Size by profile: light Go/nginx/Rust `128m`–`256m`, Node apps `384m`–`512m`, shared databases `1g`, ingress (Traefik) `384m`. Never set `memswap_limit == mem_limit` on a database — that forbids swap inside the container and turns a legitimate query spike into an instant kill.

**Docker socket:**

- Read-only (`:ro`) for monitoring tools
- Read-write for management tools

**Volume paths** — before writing a volume mount, verify the actual data path in the image: check the Dockerfile (`VOLUME`, `WORKDIR`, entrypoint args like `--dir`) on Docker Hub or the image's GitHub repo. Never guess paths.

**YAML quoting** — do not quote values unless YAML requires it. Use native unquoted booleans (`true`/`false`), including boolean Docker label flags; Compose normalizes label values for Docker. Quotes are necessary when a value must remain a string but YAML would otherwise interpret it as another type (such as a boolean or number), or when it starts with a YAML indicator character. Plain strings, hostnames, entrypoint names, and Traefik rules (including backtick expressions) do not need quotes in block context.

**Launcher script** — `./start` in repo root. Interactive TUI (Python 3 + curses, zero dependencies). Manages deploy order and auto-selects transitive dependencies. When adding a new service: add it to `ORDER`, `DEPS`, and `LABELS` in `start` — all three should list services in the same order (Layer 1 by deploy priority, Layer 2 alphabetically).

**Other:**

- **No alignment whitespace** — do not pad code with extra spaces to align values into columns.
- All services: `restart: unless-stopped`
- All services: `mem_limit` set (see **Memory limits** above)
- All services: `container_name: {name}` (avoid auto-generated names like `traefik-traefik-1`)
- `.env.example` in repo root documents all variables across all services
- Each service with local volume directories has its own `.gitignore`; root `.gitignore` covers only repo-wide ignores (`.env`)

## Keeping docs in sync

When adding or changing a service — always update `README.md`, `AGENTS.md`, and `.env.example` without being asked:

- `README.md` — full setup details, key env vars, prerequisites
- `AGENTS.md` — update the current stack table and any changed conventions
- `.env.example` — add any new env vars for the service
- `compose/hub/public/index.html` — if the service has a web UI, add it to the `layer1` or `layer2` array in the script block

When introducing a new pattern or convention (naming, file structure, gitignore approach, etc.) — add it to `AGENTS.md` immediately.

## Current stack

Layer 1 order is intentional — it reflects recommended deployment sequence (dependencies first, backups last). Do not reorder without a reason.

| Service | Subdomain | Entrypoint | Notes |
| --- | --- | --- | --- |
| Watchtower | — | — | Opt-in auto-updates for explicitly labeled containers |
| Traefik | `traefik.DOMAIN` | tailscale | Reverse proxy, TLS, dashboard |
| Vaultwarden | `vaultwarden.DOMAIN` | tailscale | Password manager |
| PostgreSQL | — | — | Shared DB, `postgres` Docker network |
| Redis | — | — | Shared cache, `redis` Docker network |
| MongoDB | — | — | Shared document DB, `mongo` Docker network |
| Dozzle | `dozzle.DOMAIN` | tailscale | Container log viewer |
| Netdata | `netdata.DOMAIN` | tailscale | Server and container metrics |
| Dockge | `dockge.DOMAIN` | tailscale | Compose stack manager |
| Uptime Kuma | `uptime.DOMAIN` | tailscale | Uptime monitoring |
| Restic | — | — | Encrypted incremental backups |
| Hub | `hub.DOMAIN` | tailscale | Static services dashboard, domain derived from URL at runtime |
| n8n | `n8n.DOMAIN` (editor) · `hooks.DOMAIN` (webhooks) | tailscale + websecure | Workflow automation, AI agents |
| OpenPanel | `op.DOMAIN` (UI) · `op-api.DOMAIN` (ingestion) | tailscale (UI) + websecure (ingestion) | Product analytics; embedded Postgres 14 + ClickHouse + Redis (BullMQ, `noeviction`) |
| PocketBase | `pb.DOMAIN` (admin) · `pb-api.DOMAIN` (API) | tailscale (admin UI) + websecure (public API) | Lightweight BaaS, SQLite, admin on the wildcard-private base host |

Services documented but not yet composed: Bugsink, CloudBeaver, Hoppscotch, Penpot, Umami.
