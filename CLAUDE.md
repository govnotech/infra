# Server Infrastructure — Agent Context

Self-hosted VPS stack. Each service lives in `compose/{name}/compose.yml`. All compose files are run from the **repo root** so the shared `.env` is auto-loaded:

```bash
docker compose -f compose/traefik/compose.yml up -d
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
  traefik.http.routers.{name}.rule: "Host(`{subdomain}.${DOMAIN}`)"
  traefik.http.routers.{name}.entrypoints: "tailscale" # or websecure
  traefik.http.routers.{name}.tls: true
  traefik.http.routers.{name}.tls.certresolver: "lets-encrypt"
  traefik.http.services.{name}.loadbalancer.server.port: { port }
```

**Image versioning:**

- Critical services (Postgres, Redis, Traefik) → pin to major version (`postgres:16`, `traefik:v3`)
- Non-critical → `latest` is fine
- [Watchtower](compose/watchtower/compose.yml) handles minor/patch updates; never jumps major versions

**Docker socket:**

- Read-only (`:ro`) for monitoring tools
- Read-write for management tools

**Other:**

- All services: `restart: unless-stopped`
- All services: `container_name: {name}` (avoid auto-generated names like `traefik-traefik-1`)
- `.env.example` in repo root documents all variables across all services
- Each service with local volume directories has its own `.gitignore`; root `.gitignore` covers only repo-wide ignores (`.env`)

## Keeping docs in sync

When adding or changing a service — always update `README.md`, `CLAUDE.md`, and `.env.example` without being asked:

- `README.md` — full setup details, key env vars, prerequisites
- `CLAUDE.md` — update the current stack table and any changed conventions
- `.env.example` — add any new env vars for the service

When introducing a new pattern or convention (naming, file structure, gitignore approach, etc.) — add it to `CLAUDE.md` immediately.
