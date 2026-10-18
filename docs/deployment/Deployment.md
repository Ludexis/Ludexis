# Ludexis Deployment Guide

Ludexis is deployed with Docker Compose. This guide covers the production stack, first sign-in, configuration, storage, updates, backups, HTTPS, monitoring and the development and demo stacks.

## Compose files

| File | Purpose |
| ---- | ------- |
| `docker-compose.yml` | Production: `db`, `redis`, `backend`, `worker`, `beat`, `frontend` and `proxy` (Caddy, the only published port). Optional `monitoring` profile: Prometheus, Node Exporter and Grafana. |
| `docker-compose.dev.yml` | Development: PostgreSQL and Redis on localhost; the application runs on the host. |
| `docker-compose.demo.yml` | Evaluation: the full stack built from source and seeded with a demo dataset. |

## Quick start

1. Copy `.env.example` to `.env` and set `GAMES_PATH` (mounted read-only at `/games`), `TZ` and `POSTGRES_PASSWORD`.
2. `docker compose up -d` (add `--build` to build from the checkout instead of pulling `ghcr.io/ludexis/ludexis-*`).
3. Open `http://<host>:8420` and sign in as `admin` / `admin`.

### First sign-in

On a database without accounts the backend creates the default administrator (`DEFAULT_ADMIN_USERNAME`, `DEFAULT_ADMIN_PASSWORD`, both `admin` unless set) and logs a warning. The account is flagged `must_change_password`:

- the sign-in page names the account while the flag is set;
- every API request from it except `/api/auth/me`, `/api/auth/change-password` and `/api/auth/logout-all` returns 403;
- the web app opens first-run setup: a new password (at least 8 characters), optionally a new username and email, the first library (`/games`), optional IGDB credentials, then the first scan.

Changing the password clears the flag and signs out every other session.

## Routing

| Path on the proxy | Goes to |
| ----------------- | ------- |
| `/api/metrics` | 404 (scraped inside the stack only) |
| `/api/*`, `/media/*`, `/healthz` | backend:8000 |
| everything else | frontend:3000 |

The frontend image is built with relative `/api` and `/media` URLs, so it works behind any hostname without rebuilding.

## Configuration (`.env`)

| Variable | Default | Meaning |
| -------- | ------- | ------- |
| `GAMES_PATH` | required | Host folder with your games, mounted read-only at `/games`. |
| `LUDEXIS_PORT` | `8420` | HTTP port of the proxy. |
| `LUDEXIS_HTTPS_PORT` | `8443` | HTTPS port of the proxy. |
| `LUDEXIS_DOMAIN` | empty | Domain for automatic HTTPS. Empty serves plain HTTP. |
| `LUDEXIS_VERSION` | `latest` | Image tag. `latest` follows the main branch; `1.2.3` or `1.2` pins a release. |
| `TZ` | `UTC` | Time zone of scheduled tasks. |
| `POSTGRES_PASSWORD` | `ludexis` | Database password inside the stack. |
| `DEFAULT_ADMIN_USERNAME` / `DEFAULT_ADMIN_PASSWORD` | `admin` / `admin` | First-start account; must change its password at first sign-in. |
| `JWT_SECRET_KEY` | generated | Leave empty to generate one into `/data/config/jwt_secret`. If set, at least 32 characters. |
| `TWITCH_CLIENT_ID` / `TWITCH_CLIENT_SECRET` | empty | IGDB credentials (also settable in Admin > Settings). |
| `STEAMGRIDDB_API_KEY` | empty | Community covers, banners and logos. |
| `PROVIDER_PLUGINS_DIR` | empty | Folder of extra metadata provider modules (`*.py`, each defining a `MetadataProvider` subclass), loaded at startup. Mount it into the backend and worker; a provider is used once added to Admin > Settings > Provider order. |
| `CORS_ORIGINS` | empty | Only needed when the web app is served from another origin. |
| `GRAFANA_PORT` / `GRAFANA_ADMIN_PASSWORD` | `3000` / `admin` | Monitoring profile only. |

## Storage

| Volume | Contents |
| ------ | -------- |
| `db_data` | PostgreSQL. |
| `ludexis_data` | `/data/artwork` (downloaded artwork) and `/data/config/jwt_secret`. |
| `caddy_data` | HTTPS certificates. |
| `prometheus_data`, `grafana_data` | Monitoring profile only. |

Games are mounted read-only and never modified; the backend (uid 1000) needs read access to them.

## Health checks

The backend answers `/healthz`, the worker answers a Celery ping, beat keeps its schedule file fresh and the frontend answers on port 3000. The proxy starts once the backend and frontend are healthy. `/api/health/db` and `/api/health/redis` report `healthy` or 503 without error details (details go to the backend log).

## Updating

```bash
docker compose pull && docker compose up -d
```

Migrations run when the backend starts (`alembic upgrade head`).

## Backups

```bash
docker compose exec -T db pg_dump -U ludexis -Fc ludexis > ludexis-db.dump
docker compose run --rm --no-deps -v "$PWD:/backup" --entrypoint tar backend czf /backup/ludexis-data.tgz -C /data .
```

Restore into a fresh stack:

```bash
docker compose up -d db
docker compose exec -T db pg_restore -U ludexis -d ludexis --clean --if-exists < ludexis-db.dump
docker compose run --rm --no-deps -v "$PWD:/backup" --entrypoint tar backend xzf /backup/ludexis-data.tgz -C /data
docker compose up -d
```

## HTTPS

Point a domain at the host and set `LUDEXIS_DOMAIN`, `LUDEXIS_PORT=80` and `LUDEXIS_HTTPS_PORT=443`. Caddy obtains and renews the certificate and redirects HTTP to HTTPS; the backend sees `X-Forwarded-Proto: https` and marks the session cookies `Secure`.

## Monitoring

```bash
docker compose --profile monitoring up -d
```

Prometheus scrapes `backend:8000/api/metrics` and Node Exporter (`deploy/prometheus.yml`). Grafana listens on `GRAFANA_PORT` and asks to replace `admin`/`admin` at first sign-in; add a Prometheus data source with the URL `http://prometheus:9090`.

## Scheduled tasks

The nightly incremental scan, metadata refresh, artwork validation and the weekly integrity verification and duplicate detection are stored in the database and edited under Admin > Scheduled Tasks. Celery beat runs a five-minute tick that starts due tasks.

## Development stack

```bash
docker compose -f docker-compose.dev.yml up -d
```

PostgreSQL (5432) and Redis (6379) listen on localhost (IPv4 and IPv6). Run the backend, worker, beat and frontend on the host as described in the README's Developer Setup; `backend/.env.example` points at these services.

## Demo stack

```bash
export JWT_SECRET_KEY=$(openssl rand -hex 32)
docker compose -f docker-compose.demo.yml up -d --build
```

The demo seeds an `admin` account (password `DEMO_ADMIN_PASSWORD`, default `Admin123!`) and sample entries, and serves the web app on port 3000 and the API on 8000. Its containers share no config volume, so the JWT secret must be passed in.

## Images

Each image is built by the CI workflow of its own repository, [ludexis-backend](https://github.com/Ludexis/ludexis-backend) and [ludexis-frontend](https://github.com/Ludexis/ludexis-frontend), checked out here as the `backend` and `frontend` submodules. The workflow runs that repository's tests, and only when they pass publishes `ghcr.io/ludexis/ludexis-backend` or `ghcr.io/ludexis/ludexis-frontend` for `linux/amd64` and `linux/arm64`: on every push to `main` (tags `main` and `latest`) and on `v*` tags (`1.2.3`, `1.2`). The frontend's build stages run on the build machine's architecture; only the copy-only runtime stage is per platform.

Both images run unprivileged: the backend as `ludexis` (uid 1000), the frontend as `node`. A new `ludexis_data` volume starts out owned by uid 1000; the games folder only needs to be readable by it.

## Troubleshooting

| Symptom | Check |
| ------- | ----- |
| Backend restarts | `docker compose logs backend`: database reachable, migrations applied, `JWT_SECRET_KEY` at least 32 characters if set. |
| Library shows Offline | The folder under `/games` is missing or empty while the library has entries; check the `GAMES_PATH` mount. |
| Jobs stay pending | `docker compose logs worker` and the worker health check. |
| Forgot the administrator password | Another administrator can reset it in Admin > Users, or on the server: `docker compose exec backend python -m scripts.reset_password <username>` (back to the default password, to be changed at the next sign-in). |
