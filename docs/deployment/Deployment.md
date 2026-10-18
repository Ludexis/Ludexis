# Ludexis Deployment Guide

Ludexis is deployed with Docker Compose. This guide covers the production stack, first sign-in, configuration, storage, updates, backups, HTTPS, monitoring, edge devices and the development stack.

## Compose files

| File | Purpose |
| ---- | ------- |
| `docker-compose.yml` | Production: `db`, `redis`, `backend`, `worker`, `beat`, `frontend` and `proxy` (Caddy, the only published port). Optional `monitoring` profile: Prometheus, Node Exporter and Grafana. |
| `docker-compose.dev.yml` | Development: PostgreSQL and Redis on localhost; the application runs on the host. |

Every container reads `.env` through `env_file`; the compose file itself only fixes the paths inside the containers. The backend, worker and beat run the same image with `command: ludexis server`, `ludexis worker` or `ludexis beat`, and their health checks call `ludexis health <role>`. `docker compose exec backend <command>` still runs any other command, such as `python -m scripts.reset_password`.

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
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | `ludexis` / `ludexis` / `ludexis` | Database account, read by the database container and by the backend, which builds its connection URL from them. Only reachable inside the stack. |
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

## Edge devices

An edge device scans drives attached to another machine, such as a laptop, and sends what it finds to this server, which keeps the only database. It is optional: libraries located on the server work exactly as before.

1. **Admin > Edge > Add edge**, give it a name. The token is shown once; download the install files for Linux or Windows right away (a new token can be issued later from the device's menu).
2. On that machine, with Docker installed:
   - Linux: `unzip ludexis-edge-linux.zip && cd ludexis-edge && sudo ./install.sh`. It installs to `/opt/ludexis-edge`, adds the `ludexis-edge` command, a udev rule and `ludexis-drive@.service`, and starts the device.
   - Windows: unzip, then in PowerShell `powershell -ExecutionPolicy Bypass -File install.ps1`. It installs to `%LOCALAPPDATA%\Ludexis\edge`, puts `ludexis-edge` on your PATH and starts the device (and Docker Desktop if needed).
3. **Admin > Libraries > Add library**: name, the folder path as that machine shows it (`/media/you/Games`, `E:\Games`), and the device as its **Location**. The device scans it at its next check-in (every 15 seconds); while it is off, the scan waits.

After its first scan a library is tied to its drive's UUID or volume serial, so it is found again under another mount point or drive letter, and moving a drive between Linux and Windows keeps it. Entries are never deleted by a scan: an unplugged drive makes the library offline, and files missing from a listing become Missing until they return.

| Command on the device | What it does |
| --------------------- | ------------ |
| `ludexis-edge start` | Start the device (does nothing if it runs) |
| `ludexis-edge restart` | Check Docker, the server, the token and the drives; map drive letters (Windows) or extra folders (Linux, `EDGE_ROOTS` in `.env`); recreate the container; print a summary |
| `ludexis-edge status` | The summary without restarting |
| `ludexis-edge stop`, `logs`, `update` | Stop it; follow its log; pull the newest image and restart |

- **Linux, automatic:** plugging in a drive that holds a library mounts it read-only at `/mnt/ludexis/<UUID>` (unless the desktop mounted it first) and starts the device; with **Scan on connect** on (the default) the server queues the scan. Library folders must be under `/media`, `/mnt` or `/run/media`, or a folder listed in `EDGE_ROOTS`.
- **Windows:** Docker Desktop only sees drives that existed when the container started, so run `ludexis-edge restart` after plugging a drive in. Mapped network drives are not visible to Docker Desktop.
- **Metadata here:** with this switch on, the device looks up metadata for its new games itself and the server stores the results, for sources that refuse the server's address. Keys for IGDB go in the device's `.env`; extra providers in its `providers/` folder with `PROVIDER_PLUGINS_DIR=/providers`.
- Keep the device on the server's release (`LUDEXIS_VERSION` in its `.env`, then `ludexis-edge update`); Admin > Edge flags a device that runs another release.

## Images

Each image is built by the CI workflow of its own repository, [ludexis-backend](https://github.com/Ludexis/ludexis-backend) and [ludexis-frontend](https://github.com/Ludexis/ludexis-frontend), checked out here as the `backend` and `frontend` submodules. The workflow runs that repository's tests, and only when they pass publishes `ghcr.io/ludexis/ludexis-backend` or `ghcr.io/ludexis/ludexis-frontend` for `linux/amd64` and `linux/arm64`: on every push to `main` (tags `main` and `latest`) and on `v*` tags (`1.2.3`, `1.2`). A `v*` tag also creates a GitHub release in that repository; tag all three repositories with the same version, the main one's release text coming from `docs/release/RELEASE_NOTES.md`. The frontend's build stages run on the build machine's architecture; only the copy-only runtime stage is per platform.

Both images run unprivileged: the backend as `ludexis` (uid 1000), the frontend as `node`. A new `ludexis_data` volume starts out owned by uid 1000; the games folder only needs to be readable by it.

## Troubleshooting

| Symptom | Check |
| ------- | ----- |
| Backend restarts | `docker compose logs backend`: database reachable, migrations applied, `JWT_SECRET_KEY` at least 32 characters if set. |
| Library shows Offline | The folder under `/games` is missing or empty while the library has entries; check the `GAMES_PATH` mount. |
| Jobs stay pending | `docker compose logs worker` and the worker health check. |
| Edge device shows Offline | On that machine: `ludexis-edge status` (server reachable, token accepted, drives found). A refused token needs new install files from Admin > Edge. |
| Edge library shows "not mapped" (Windows) | Docker Desktop only sees drive letters that existed when the container started: run `ludexis-edge restart`. |
| Forgot the administrator password | Another administrator can reset it in Admin > Users, or on the server: `docker compose exec backend python -m scripts.reset_password <username>` (back to the default password, to be changed at the next sign-in). |
