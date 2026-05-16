# Server Infrastructure

> Minimum specs: **4 vCPU / 8 GB RAM / 160 GB NVMe SSD**, Ubuntu 22.04 LTS or Debian 12.

![Hub dashboard](screenshot.webp)

## Table of Contents

- [Quick Start](#quick-start)
- [Access Model](#access-model)
- [Layer 0 — Network](#layer-0--network)
  - [Cloudflare](#cloudflare)
  - [Tailscale](#tailscale)
- [Layer 1 — Infrastructure](#layer-1--infrastructure)
  - [Watchtower](#watchtower)
  - [Traefik](#traefik)
  - [PostgreSQL (shared)](#postgresql-shared)
  - [Dozzle](#dozzle)
  - [Netdata](#netdata)
  - [Dockge](#dockge)
  - [Uptime Kuma](#uptime-kuma)
  - [Hub](#hub)
- [Layer 2 — Services](#layer-2--services)
  - [OpenPanel](#openpanel)
  - [PocketBase](#pocketbase)
- [Layer 3 — Applications](#layer-3--applications)

---

## Quick Start

1. Clone the repository onto the server.
2. Copy `.env.example` to `.env` and fill in the values:

   ```bash
   cp .env.example .env
   ```

3. Launch services with the interactive launcher:

   ```bash
   ./start
   ```

   Use `↑`/`↓` to navigate, `SPACE` to select, `ENTER` to launch. Dependencies are resolved automatically — selecting n8n, for example, auto-selects Traefik and PostgreSQL.

   **Manual alternative** — run each service directly from the repo root (`--env-file .env` is required because Docker Compose v2 resolves `.env` relative to the compose file, not cwd):

   ```bash
   docker compose --env-file .env -f compose/watchtower/compose.yml up -d
   docker compose --env-file .env -f compose/traefik/compose.yml up -d
   # …
   ```

**Image versioning:** Critical services (PostgreSQL, Redis) are pinned to a major version (e.g. `postgres:16`) — [Watchtower](#watchtower) updates patch/minor releases but never jumps to a new major. Non-critical services (Dozzle, Watchtower itself) use `latest`.

Private services route through the `tailscale` Traefik entrypoint — DNS for their subdomains points to `TRAEFIK_TAILSCALE_IP` (Cloudflare grey cloud), so they are only reachable from within the Tailscale network.

---

## Access Model

Three access tiers. Each service belongs to exactly one.

| Service | Tier | How |
| ----------------------------- | ----------------- | -------------------------------------------------------- |
| SaaS front-ends, public sites | Public (proxied) | Cloudflare orange cloud → Traefik |
| OpenPanel event ingestion | Public (proxied) | Cloudflare orange cloud → Traefik |
| API backends | Public (DNS-only) | Cloudflare grey cloud → Traefik |
| All admin / dashboard UIs | Private | Cloudflare grey cloud → Traefik (`tailscale` entrypoint) |

**Cloudflare proxy (orange cloud)** — hides origin IP, adds CDN and DDoS protection.
**Cloudflare DNS-only (grey cloud)** — plain DNS, no proxy. Used for webhooks, API backends, and private services (DNS points to Tailscale IP).
**Tailscale** — WireGuard VPN mesh. Private services use a dedicated Traefik entrypoint bound to the Tailscale IP — only reachable from within the tailnet.

---

## Layer 0 — Network

### Cloudflare

`Layer 0` · [dash.cloudflare.com](https://dash.cloudflare.com) · [Docs](https://developers.cloudflare.com/dns/)

DNS provider, CDN, and DDoS protection. Configured entirely in the dashboard — no server-side installation.

**Setup:**

1. Add your domain to Cloudflare and point nameservers.
2. Create an `A` record pointing your apex/subdomain to the VPS IP.
3. Set orange cloud (proxied) for public-facing services; grey cloud (DNS-only) for webhooks, API backends, and private services (point to `TRAEFIK_TAILSCALE_IP`).
4. In **SSL/TLS → Overview**, set mode to **Full (strict)**.
5. Create an API token for `CF_DNS_API_TOKEN`: **My Profile → API Tokens → Create Token → Edit zone DNS** (use the template) → set **Zone Resources** to your domain → **Create Token**.

### Tailscale

`Layer 0` · [login.tailscale.com/admin](https://login.tailscale.com/admin) · [Docs](https://tailscale.com/kb/) · [GitHub](https://github.com/tailscale/tailscale)

WireGuard-based VPN mesh. Connects the VPS, your Mac, and any other devices. Used to restrict access to private services — Traefik's `tailscale` entrypoint is bound to the Tailscale IP, so private subdomains are only reachable from within the tailnet.

**Setup:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate the server (use an auth key from the Tailscale admin console)
tailscale up --authkey=<auth-key>
```

Enable **MagicDNS** in the admin console under DNS settings.

---

## Layer 1 — Infrastructure

_Sorted by deployment priority — each service may be a prerequisite for those that follow._

### Watchtower

`Layer 1` · [Docs](https://containrrr.dev/watchtower/) · [GitHub](https://github.com/containrrr/watchtower)

**Prerequisites:** Docker.

**Deploy:** [`compose/watchtower/compose.yml`](compose/watchtower/compose.yml)

Automatically pulls and restarts containers when new images are published. Deploy first — covers all services that follow.

**Required in `.env`:** `WATCHTOWER_SCHEDULE` — see [`.env.example`](.env.example).

By default, watches all containers. Use `--label-enable` to opt-in specific containers instead.

### Traefik

`Layer 1` · [Docs](https://doc.traefik.io/traefik/) · [GitHub](https://github.com/traefik/traefik)

**Prerequisites:** [Cloudflare](#cloudflare) API token, [Tailscale](#tailscale), `traefik` Docker network.

**Deploy:** [`compose/traefik/compose.yml`](compose/traefik/compose.yml)

Reverse proxy for all public-facing services. Handles TLS termination via Let's Encrypt (DNS-01 challenge through [Cloudflare](#cloudflare)). Routes requests to containers using Docker labels.

```bash
docker network create traefik
```

**Required in `.env`:** `CF_DNS_API_TOKEN`, `ACME_EMAIL`, `TRAEFIK_PUBLIC_IP`, `TRAEFIK_TAILSCALE_IP`, `DOMAIN` — see [`.env.example`](.env.example).

Two entrypoints:

- `websecure` — bound to `TRAEFIK_PUBLIC_IP:443`, for public services.
- `tailscale` — bound to `TRAEFIK_TAILSCALE_IP:443`, for private services. DNS for private subdomains must point to `TRAEFIK_TAILSCALE_IP` (Cloudflare grey cloud).

Any service must join the `traefik` network and carry `traefik.enable=true` labels. Public services use `entrypoints: websecure`, private services use `entrypoints: tailscale`.

### PostgreSQL (shared)

`Layer 1` · [Docs](https://www.postgresql.org/docs/) · [GitHub](https://github.com/postgres/postgres)

**Prerequisites:** Docker.

**Deploy:** [`compose/postgres/compose.yml`](compose/postgres/compose.yml)

Single PostgreSQL instance shared by [Umami](#umami), [Hoppscotch](#hoppscotch), [Penpot](#penpot), and any other services that need a relational DB. Each service gets its own database and user.

**Required in `.env`:** `POSTGRES_USER`, `POSTGRES_PASSWORD` — see [`.env.example`](.env.example).

Create per-service databases after first start:

```bash
docker exec -it postgres psql -U $POSTGRES_USER
```

```sql
CREATE DATABASE umami;
CREATE USER umami_user WITH PASSWORD '...';
GRANT ALL PRIVILEGES ON DATABASE umami TO umami_user;
```

### Dozzle

`Layer 1` · [Docs](https://dozzle.dev/guide/what-is-dozzle) · [GitHub](https://github.com/amir20/dozzle)

**Prerequisites:** [Traefik](#traefik) (private UI).

**Deploy:** [`compose/dozzle/compose.yml`](compose/dozzle/compose.yml)

Real-time log viewer for all running containers. Read-only Docker socket access.

### Netdata

`Layer 1` · [Docs](https://learn.netdata.cloud) · [GitHub](https://github.com/netdata/netdata)

**Prerequisites:** [Traefik](#traefik) (private UI).

**Deploy:** [`compose/netdata/compose.yml`](compose/netdata/compose.yml)

Server and per-container metrics with zero configuration. Auto-discovers running containers. ML-based anomaly detection and alerting.

Configure a notification channel (Telegram, email) in `netdata.conf` or via the UI to receive alerts.

### Dockge

`Layer 1` · [Docs](https://github.com/louislam/dockge) · [GitHub](https://github.com/louislam/dockge)

**Prerequisites:** [Traefik](#traefik) (private UI).

**Deploy:** [`compose/dockge/compose.yml`](compose/dockge/compose.yml)

Visual manager for Docker Compose stacks. Useful for spinning up temporary stacks without SSH.

Stacks are stored in `compose/dockge/stacks/` — no configuration needed.

### Uptime Kuma

`Layer 1` · [Docs](https://github.com/louislam/uptime-kuma/wiki) · [GitHub](https://github.com/louislam/uptime-kuma)

**Prerequisites:** [Traefik](#traefik) (private UI).

**Deploy:** [`compose/uptime-kuma/compose.yml`](compose/uptime-kuma/compose.yml)

Uptime monitoring for external URLs. Sends alerts via Telegram or email when a service goes down.

### Hub

`Layer 1` · [GitHub](https://github.com/nginx/nginx) · [`compose/hub/compose.yml`](compose/hub/compose.yml)

**Prerequisites:** [Traefik](#traefik) (private UI).

**Deploy:** [`compose/hub/compose.yml`](compose/hub/compose.yml)

Private homepage listing all self-hosted services with links. Served by nginx:alpine from a single static HTML file — no build step, no external dependencies. The page derives the base domain from `window.location.hostname` at runtime, so no env vars are needed beyond `DOMAIN` (already required by Traefik).

All service URLs are displayed as `{subdomain}.DOMAIN` and open in the same tab on click. The service list is hardcoded in the `layer1` / `layer2` arrays inside `compose/hub/index.html` — update it manually when adding or removing a service.

---

## Layer 2 — Services

_Sorted alphabetically._

### OpenPanel

`Layer 2` · [Docs](https://openpanel.dev/docs) · [GitHub](https://github.com/Openpanel-dev/openpanel)

**Prerequisites:** [Traefik](#traefik) (event ingestion), [Tailscale](#tailscale) (dashboard).

**Deploy:** [`compose/openpanel/compose.yml`](compose/openpanel/compose.yml)

Open-source product & web analytics — funnels, retention, user journeys, event tracking. Mixpanel/PostHog alternative. Bundles its own Postgres, Redis (BullMQ queue) and ClickHouse — not connected to the shared instances because OpenPanel pins exact versions and runs Redis with `noeviction` for job durability.

Single host `op.DOMAIN`:

- Dashboard — private, tailscale entrypoint only.
- `/api` (event ingestion) — public, both `websecure` and `tailscale` entrypoints. Traefik strips the `/api` prefix before forwarding to `op-api`.

**Required in `.env`:** `OPENPANEL_COOKIE_SECRET`, `OPENPANEL_ALLOW_REGISTRATION` — see [`.env.example`](.env.example). Generate the cookie secret:

```bash
openssl rand -hex 32
```

**Bootstrap (one-time):**

1. Start with `OPENPANEL_ALLOW_REGISTRATION=true`, open `https://op.DOMAIN` from within the tailnet, create the first account.
2. Set `OPENPANEL_ALLOW_REGISTRATION=false` in `.env` and restart openpanel (email-based invitations are disabled — only existing accounts can sign in).

**DNS:** `op.DOMAIN` needs an `A` record on `TRAEFIK_TAILSCALE_IP` (grey cloud) for the dashboard. For public event ingestion from browsers, add a second `A` record on `TRAEFIK_PUBLIC_IP` (orange cloud) — Traefik picks the right entrypoint automatically.

### PocketBase

`Layer 2` · [Docs](https://pocketbase.io/docs/) · [GitHub](https://github.com/pocketbase/pocketbase)

**Prerequisites:** [Traefik](#traefik) (public API), [Tailscale](#tailscale) (admin UI).

**Deploy:** [`compose/pocketbase/compose.yml`](compose/pocketbase/compose.yml)

Lightweight BaaS: SQLite, built-in auth, realtime subscriptions, file storage, admin UI. Single binary, ~30 MB RAM. Use for MVPs and small projects.

---

## Layer 3 — Applications

Custom applications deployed on this server. Each app connects to shared infrastructure ([Traefik](#traefik), [PostgreSQL](#postgresql-shared)) defined in Layers 0–2.

_This layer is out of scope for this repository — each application lives in its own repository with its own compose configuration._
