# Server Infrastructure — Agent Context

Self-hosted VPS stack. Each service lives in `compose/{name}/compose.yml`. All compose files are run from the **repo root** with `--env-file .env` to load the shared env file (Docker Compose v2 loads `.env` from the compose file's directory, not cwd):

```bash
docker compose --env-file .env -f compose/watchtower/compose.yml up -d
docker compose --env-file .env -f compose/traefik/compose.yml up -d
```

## Architecture

Two-tier access model via Traefik:

- **Public** (`websecure` entrypoint, `TRAEFIK_PUBLIC_IP:443`) — Cloudflare orange/grey cloud → Traefik
- **Private** (`tailscale` entrypoint, `TRAEFIK_TAILSCALE_IP:443`) — DNS points to Tailscale IP (Cloudflare grey), only reachable within tailnet

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

**Image versioning:**

- Critical services (Postgres, Redis, Traefik) → pin to major version (`postgres:16`, `traefik:v3`)
- Non-critical → `latest` is fine
- [Watchtower](compose/watchtower/compose.yml) handles minor/patch updates; never jumps major versions

**Docker socket:**

- Read-only (`:ro`) for monitoring tools
- Read-write for management tools

**Volume paths** — before writing a volume mount, verify the actual data path in the image: check the Dockerfile (`VOLUME`, `WORKDIR`, entrypoint args like `--dir`) on Docker Hub or the image's GitHub repo. Never guess paths.

**YAML quoting** — do not quote values unless YAML requires it. Quotes are only necessary when the value would otherwise be misinterpreted: booleans (`true`/`false`), numbers, or strings starting with YAML indicator characters. Plain strings, hostnames, entrypoint names, and Traefik rules (including backtick expressions) do not need quotes in block context.

**Launcher script** — `./start` in repo root. Interactive TUI (Python 3 + curses, zero dependencies). Manages deploy order and auto-selects transitive dependencies. When adding a new service: add its folder name to `ORDER` and its deps to `DEPS` in `start`.

**Other:**

- **No alignment whitespace** — do not pad code with extra spaces to align values into columns.
- All services: `restart: unless-stopped`
- All services: `container_name: {name}` (avoid auto-generated names like `traefik-traefik-1`)
- `.env.example` in repo root documents all variables across all services
- Each service with local volume directories has its own `.gitignore`; root `.gitignore` covers only repo-wide ignores (`.env`)

## Keeping docs in sync

When adding or changing a service — always update `README.md`, `CLAUDE.md`, and `.env.example` without being asked:

- `README.md` — full setup details, key env vars, prerequisites
- `CLAUDE.md` — update the current stack table and any changed conventions
- `.env.example` — add any new env vars for the service
- `compose/hub/index.html` — if the service has a web UI, add it to the `layer1` or `layer2` array in the script block

When introducing a new pattern or convention (naming, file structure, gitignore approach, etc.) — add it to `CLAUDE.md` immediately.

## Current stack

Layer 1 order is intentional — it reflects recommended deployment sequence (dependencies first, backups last). Do not reorder without a reason.

| Service | Subdomain | Entrypoint | Notes |
| --- | --- | --- | --- |
| Watchtower | — | — | Auto-updates containers |
| Traefik | `traefik.DOMAIN` | tailscale | Reverse proxy, TLS, dashboard |
| PostgreSQL | — | — | Shared DB, `postgres` Docker network |
| Dozzle | `dozzle.DOMAIN` | tailscale | Container log viewer |
| Netdata | `netdata.DOMAIN` | tailscale | Server and container metrics |
| Dockge | `dockge.DOMAIN` | tailscale | Compose stack manager |
| Uptime Kuma | `uptime.DOMAIN` | tailscale | Uptime monitoring |
| PocketBase | `pb.DOMAIN` | tailscale (admin UI) + websecure (API) | Lightweight BaaS, SQLite, admin UI blocked on public |
| Hub | `hub.DOMAIN` | tailscale | Static services dashboard, domain derived from URL at runtime |
