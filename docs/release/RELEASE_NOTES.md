# Ludexis Release Notes

## v1.2.0 (2026-10-18)

### Edge devices

- Another machine, such as a laptop, can scan drives attached to it and send the listings to the server, which keeps the only database. Add one under **Admin → Edge**, download its install files for Linux or Windows, and run the installer.
- Each library has a **Location**: the server or one edge device. A scan of an edge library waits until that device checks in.
- After the first scan a library is tied to its drive's UUID or serial, so it is found again under another mount point or drive letter.
- Linux: a known drive is mounted read-only and scanned when it is plugged in. Windows: the `ludexis-edge` command (`start`, `restart`, `stop`, `status`, `logs`, `update`) maps drives by serial.
- Optionally an edge device looks up metadata from its own network.
- A server without edge devices works as before.

### Upgrading

- Each container now runs `ludexis server`, `ludexis worker` or `ludexis beat`, and reads `.env` through `env_file`. Download the new `docker-compose.yml`; an existing `.env` keeps working. `POSTGRES_USER` and `POSTGRES_DB` default to `ludexis`.
- The demo stack (`docker-compose.demo.yml`) was removed.
- Migrations run on start; they add the edge device table and new library and job columns. To pin this release, set `LUDEXIS_VERSION=1.2.0` in `.env`.

### Fixes

- Changing a cover no longer fails when a metadata source returns no details.

## v1.1.0 (2026-10-18)

Ludexis is now split into three repositories under the [Ludexis organization](https://github.com/Ludexis): this one for deployment and documentation, plus [ludexis-backend](https://github.com/Ludexis/ludexis-backend) and [ludexis-frontend](https://github.com/Ludexis/ludexis-frontend), checked out as submodules.

### Upgrading

Images moved to `ghcr.io/ludexis/ludexis-backend` and `ghcr.io/ludexis/ludexis-frontend`. Download the new `docker-compose.yml`, or change the two `image:` lines, then run `docker compose pull && docker compose up -d`. Migrations run on start. To pin this release, set `LUDEXIS_VERSION=1.1.0` in `.env`.

### Library and versions

- Files of the same game under different names, versions or episodes appear as one card with a Versions tab, also when they were matched on different metadata sources.
- Each file can be identified on its own; fixing one leaves its other versions alone unless you choose to change them all.
- Scan a single library on demand; progress moves while archives are hashed.
- Better version parsing: dotted and dash-separated versions, and resolution and release-status words dropped from titles.
- Favourites page, plus favourite and completed marks on game pages and library cards.

### Metadata

- Extra metadata providers can be loaded from a folder (`PROVIDER_PLUGINS_DIR`).
- Provider warnings, such as an expired sign-in or a rate limit, show in the top bar.
- Interactive search across every source from the Identify dialog.
- IGDB covers at double resolution; SteamGridDB as a community artwork source.

### Artwork

- Art is stored as WebP under versioned names, so browsers cache it.
- AVIF images are accepted; AVIF and GIF covers are fitted to the card and kept still.
- Off-shape and letterboxed covers are cropped to the card, preferring true portrait art.
- Franchise images are picked automatically, or chosen or uploaded by editors.
- Artwork validation re-downloads missing screenshots.

### Access and security

- **Hide game** in a game's menu marks every version restricted, so only accounts allowed restricted games see it.
- Per-user access to libraries and restricted games, managed from the users panel.
- A default administrator is created on first start and must change its password first.
- Browser sessions in httpOnly cookies, refresh tokens stored as SHA-256 digests, rate-limited sign-in, and fixes that stop user managers granting rights they do not hold.
- Artwork is served only for games, collections and franchises the user can see.

### Home and admin

- Featured slideshow on the home page, with a fixed height and fuller rediscover row.
- Needs-attention alerts on the admin dashboard; review queue entries can be marked resolved.
- Faster library totals and queue stats.

### Fixes

- A redelivered task no longer restarts a job that already ended.
- Deleted games stay deleted on rescan; re-added libraries relink their entries.
- The scheduled artwork check no longer fails on an undefined viewer.

## v1.0.0 (2026-08-04)

Initial feature-complete release of Ludexis, a self-hosted game archive
management platform.

### Archive & Library

- Library management with configurable scan paths and incremental scans
- Archive entry catalog with cover, banner, and logo artwork
- Duplicate detection and resolution workflows
- Archive screenshots gallery (upload + metadata-driven)
- Bulk actions on the library (select, tag, move, delete)

### Metadata & Enrichment

- Automatic metadata matching against provider APIs (Steam, IGDB, GOG)
- Manual metadata editing with provider source badges
- Metadata confidence scoring and conflicts panel
- Metadata history and full audit trail
- Artwork quality indicators, comparison dialog, and version history

### Collections, Tags & Taxonomies

- Collections with custom covers and recommended entries
- Tag, developer, publisher, genre, and franchise management
- Relationship visualization between entries and franchises
- Advanced search with saved searches and saved search workflows

### Administration

- User management with role-based access control
- Effective permissions panel and permission matrix
- Audit log viewer with activity filtering
- Job history, reports, and background task analytics
- Live operational dashboard with trend visualization
- User analytics dashboard (login frequency, permissions, activity)
- Monitoring page with Prometheus metrics and Grafana integration
- Artwork administration (upload, replacement, deletion)
- Settings and metadata policy configuration

### Platform & UX

- JWT authentication with refresh tokens and auto-logout on expiry
- Role-aware navigation and protected routes
- Pagination across list views
- Responsive layout with mobile navigation
- Skip-to-content links and screen-reader labelling
- Motion-safe animations and micro-interactions
- Real-time background task polling

### Deployment

- Docker Compose stack (PostgreSQL, Redis, backend, worker, monitoring)
- Standalone Next.js frontend image
- One-command demo stack pre-seeded with a demo dataset
- `/media` static serving for artwork assets
- Deployment guide, architecture docs, and API guide

### Quality

- Backend: automated API test suite (140 tests)
- Frontend: unit/integration suite (210 tests across 37 files)
- Playwright end-to-end tests across Chromium, Firefox, and WebKit
- Visual regression screenshots
- TypeScript strict typecheck and lint clean

### Demo Credentials

The demo stack seeds an administrator account:

| Username | Password |
| -------- | -------- |
| `admin`  | `admin`  |

**Change this password immediately in any real deployment.**
