# Knowledge Graph State

Status, coverage, known gaps, and findings. Update at the end of every change to the graph.

| | |
|-|-|
| Graph version | v2 (schema in `_schema.md`) |
| Verified at commit | `421e9c7` |
| Last verified | 2026-10-11 |
| Validator | `python docs/knowledge/queries/validate.py`, must exit 0 |
| Size | 773 entities, 3,110 relationships |

---

## Project Summary

**Ludexis** is a self-hosted catalog for game archives (ZIP, RAR, 7z, installers, ROMs). It scans library folders, detects duplicates by hash, groups versions and episodes, enriches entries with metadata from VNDB, IGDB and Steam (tags and franchises included), manages artwork, and organizes entries into collections, tags, developers, publishers, and franchises. Libraries on removable drives go offline without losing data and are rescanned when they return. Scheduled tasks are edited in the admin UI. Access is controlled by JWT auth (httpOnly cookies for the browser, rate-limited login) with RBAC and audit logging. The root `docker-compose.yml` runs the whole service behind one port; optional edge devices (a laptop, another PC) scan their own drives and send the listings to the server, which keeps the only database.

| Part | Stack |
|------|-------|
| Backend | Python, FastAPI 0.143, SQLAlchemy 2.0, Pydantic 2.14, Celery 5.4, PostgreSQL 16, Redis |
| Frontend | Next.js 16.3, React 19, TypeScript 5.7, Tailwind 4, Radix/shadcn UI |
| Tests | pytest (backend), Vitest and Playwright (frontend) |
| Infra | Docker Compose (production with an optional monitoring profile, development; every backend container runs `ludexis <role>` and reads `.env` through `env_file`), edge device compose files served by the server, Caddy, Prometheus, Grafana, GitHub Actions (backend and frontend CI, images published after both pass) |

Recent work (from git history): frontend-to-backend integration (real API client, auth flow, admin pages gated on superuser), server-side search and pagination with entry counts, hash-based duplicate detection, auth on the media route, CORS fixes.

---

## Coverage

| Area | In graph | Notes |
|------|----------|-------|
| Backend routers and routes | Yes | 21 routers + `main.py`; 96 routes with `full_path`, permissions, request/response schemas |
| Services, repositories, models, schemas, providers | Yes | Public methods and line ranges verified |
| Celery tasks and beat schedule | Yes | |
| Database tables, columns, enums, migration | Yes | Association tables are `DBTable` with `is_association` |
| `app/core`, `app/utils`, `app/db`, `main.py`, seed scripts | Yes | File-level `Module` entities |
| Frontend pages, layout, components, hooks, context | Yes | Auth guard per page, render tree, API calls |
| Frontend API client layer | Yes | Every API function mapped to its backend route |
| Frontend types | Yes | `MIRRORS` edges to backend schemas with field mismatches |
| Tests | Yes, file level | 36 backend, 42 frontend test files |
| Docker, Compose, monitoring, CI config | No | See `docs/deployment/Deployment.md` (root `docker-compose.yml`, `deploy/Caddyfile`; each submodule's `.github/workflows/ci.yml` builds its image) |
| Individual settings fields usage | No | Fields listed on `module:app.core.config`; use grep for usage (ADR-013) |
| CSS, static assets, `components.json` | No | Not useful for navigation |

---

## Findings in the Code

These surfaced while the graph was built. They are recorded here, not fixed (ADR-017). Fixed findings move to the change log.

### Bugs

None open.

### Integration gaps

- `/api` routes with no frontend caller: `POST /auth/token` (OAuth2 form login for tooling) and `POST /archive-entries/` (entries are created by scans).

### Dead code and duplication

- Unused backend schemas: `ArtworkValidationResult`, `TokenPayload`, `JobStartRequest`.
- `ArchiveEntryService._assign_relations` sets `related_entries` through the ORM, which inserts `archive_entry_relations` rows without the NOT NULL `relationship_type`, so a create or update with `related_entry_ids` fails.
- The `franchise_entries` table is created by the migration and imported in `models/archive_entry.py`, but no ORM relationship uses it.
- `PermissionName` and `RoleName` enums are not used by any column (permission and role names are stored as strings).

### Stubs

- `ManualProvider` returns empty results; VNDB, IGDB and Steam provide metadata.
- `archive_entries.storage_device` is never written.

### Repository hygiene

- The Celery beat schedule files (`backend/celerybeat-schedule.bak`, `.dat`, `.dir`) are runtime state that was tracked in git; the `.gitignore` entry `celerybeat-schedule` did not match the extensions. Fixed: pattern is now `celerybeat-schedule*` and the files are untracked.

---

## Known Limits of the Graph

- `FLOWS_TO` edges are derived (route -> service -> repository/table) and marked `inferred`.
- `TESTS` edges for backend API tests come from the routes and classes the tests exercise, so some are indirect.
- API functions that only wrap other API functions have no `CALLS_ENDPOINT` edge of their own.
- Line numbers are exact at the verified commit; after code changes rely on the validator's staleness warning.

---

## Maintenance Procedure

1. Change code.
2. Update the affected records in the matching `graph/*.jsonl` files and domain/flow docs; set `last_verified_at` and `commit_hash`.
3. Run `validate.py` until it exits 0.
4. Update the counts in `index.md` from `validate.py --stats-json` and the table at the top of this file.

## Change Log

| Date | Change |
|------|--------|
| 2026-09-07 | v1: backend-only graph (332 entities, 861 relationships) |
| 2026-09-12 | v2: audit against code. Fixed wrong and missing backend edges (task consumers, route-to-service calls, provider dependency), merged duplicate association tables, corrected domain membership, added route `full_path`, request/response schema edges, core/utils modules, migration. Added the full frontend graph (pages, components, hooks, context, API client mapped to backend routes, types mirrored to schemas, tests). Rewrote domain and flow docs from code (removed non-existent method names and wrong storage and hashing claims). Added `validate.py` and `kg.py`. Removed em dashes and mojibake. |
| 2026-09-27 | Bug fixes `80de6bf`..`7b1ba80`: password reset field, folder hashing in scans, real tasks for `METADATA_REFRESH`/`DUPLICATE_DETECTION`/`INTEGRITY_VERIFICATION` (`run_job` removed; `verify_integrity_task`, `detect_duplicates_task` added), partial `PATCH` for archive entries, `/media` token via query parameter, setup page wired to `setupApi`. Graph and docs patched. |
| 2026-10-01 | Pipeline and release work: one entry per game folder, per-file scan errors, changed-file updates, move/copy handling, real progress and `retry_count` through the shared `run_job` (`backend/app/tasks/job_runner.py`); scans queue an enrichment job (`EnrichmentService`) that auto-matches across IGDB and Steam (`title_similarity`, 0.85/0.70) and fills missing artwork; nightly refresh covers never-attempted entries; artwork job repairs instead of touching `verification_status`; scoped media tokens replace access tokens in `/media` URLs; `AuditLogService` and `MatchingService` removed and audit gaps filled; taxonomy CRUD UI (`TaxonomyFormDialog`); unused UI primitives and hooks removed; Celery beat in both Compose stacks. Fixed findings removed from this file. Graph and docs patched. |
| 2026-10-05 | Self-hosted service work: folder-name parser (installments, bracket versions, noise, DLsite codes); `GroupingService` (versions under one primary, auto collections for episodes) with grouped search and a versions endpoint; `VNDBProvider`, provider order setting, tag and franchise import (`tags.origin`); library `status`/`relative_path` with offline detection, reconnect rescans and re-linking; `scheduled_tasks`/`system_settings` with the five-minute `scheduler_tick_task`, admin APIs and the Scheduled Tasks page; httpOnly cookie sessions with CSRF header and login rate limit (media tokens removed); stale-job sweep; auto-download as a job; homelab compose, Caddy proxy, generated JWT secret and image workflow. Four migrations. Fixed findings removed. Graph and docs patched. |
| 2026-10-11 | Franchise images: chosen (copied into `franchises/<id>/`, picked from the games' art or SteamGridDB, or uploaded) or automatic from the first released game's banner (`image_path`); franchises page shows cards; admin overview loads library totals apart from the Celery worker queries, and queue stats wait only for known workers. |
| 2026-10-11 | Default administrator created on first start and forced to change its password (`users.must_change_password`, `app.services.bootstrap`, `scripts.reset_password`; `/setup/initialize` and `seed_rbac.py` removed); media served only for visible games; user managers cannot grant rights they lack; refresh tokens stored as SHA-256; unused provider artwork endpoint and the `notes`, `ratings`, `metadata_sources` tables removed; FastAPI 0.143 and Next.js 16.3. Graph resynced from the code: frontend route groups, removed and added files, line numbers, test counts; validator module list covers scripts and function-only services, and route-group layouts keep the group in their id. |
| 2026-10-06 | Company names merged by `company_key` (`name_key` on developers and publishers, migration `b2e7c4a9d156` merges duplicates); every enabled provider supplements the primary match (`_find_same_game`); Identify (`search_providers`, `POST /archive-entries/{id}/identify`, `IdentifyDialog`, artwork job `replace` mode); EPISODE relations for series; reconnected libraries verified at once; `VersionList`; change password and log out everywhere (`users.sessions_revoked_at`, migration `d8f3a1c6e297`, `AccountSecurityCard`); GOG provider removed; HTTPS domain option, health checks, arm64 images and backup docs for the homelab stack. Graph and docs patched. |
| 2026-10-12 | Artwork storage: `optimize_artwork` (`app.utils.artwork`), called by `StorageService.save`, stores banners, screenshots and franchise images as WebP q85 (banners at most 1920 px wide), logos as lossless WebP and covers unchanged, under content-versioned names (`.v<hash>`); `/media` caches versioned names for a year (`immutable`); `ArtworkService.optimize_stored_artwork` converts older files in the artwork validation job; screenshot download and deduplication use the path `save` returns. Graph patched. |
| 2026-10-12 | IGDB covers are fetched at double size (`t_cover_big_2x`, 528x704) in `IGDBProvider.get_details`; existing covers change when artwork is refreshed. |
| 2026-10-12 | Scan progress counts 1 MiB hash chunks (`ScannerService._progress_weight`, `HASH_CHUNK`), so the bar moves while a large archive is hashed. `POST /api/scan/full` and `/scan/incremental` take an optional `library_id` (passed to the task as a kwarg) that scans only that library, enabled or not; the admin Libraries page has a Scan button per row and Full scan in the row menu. |
| 2026-10-19 | Edge devices: `edge_devices` table, `libraries.edge_id`/`drive_*` (path unique per machine), `job_history.edge_id`/`library_id` (migration `c6d1f8a2e940`); `EdgeService` and the `/edge-devices` (admin) and `/edge/agent` (device token) routers; `ScannerService.ingest_report` stores a device's listing through the shared `_ingest_batches`; edge jobs never block or get failed by the server worker; optional metadata lookups on the device (`EdgeService.apply_metadata`); `/admin/edge` page and a Location field on libraries. Compose: `x-backend` and the demo stack removed, `bin/ludexis` entry point, `DATABASE_URL` and the Celery URLs derived from `POSTGRES_*` and `REDIS_URL`. New domain `domain:edge` and flow `edge-scan`. Graph patched. |
| 2026-10-19 | Edge scans: automatic scans (no user: autoscan, schedules) hash nothing and send new or resized files as `pending`, counted in `libraries.pending_changes` (migration `d2f7a9c41b63`); a reconnected library is no longer re-hashed in full; the Libraries page asks to check pending files. Windows `ludexis-edge restart` mounts drives connected after Docker Desktop started and reads every library. Graph patched. |
| 2026-10-12 | Extra metadata providers: `load_plugin_providers` (`app.providers.plugins`) imports every `MetadataProvider` subclass from the `*.py` modules in `PROVIDER_PLUGINS_DIR`; they join `MetadataService`, `match_provider_names()` (replaces `MATCH_PROVIDERS`) and, with `restricted = True` on the class, `restricted_providers()` (replaces `RESTRICTED_PROVIDERS`). Off until added to `provider_order`. Graph patched; the validator tracks function-only provider modules. |
| 2026-10-13 | File-name parser: dash-separated versions (`DASHED_VERSION`, `0-20-16`, tried only when there is no dotted version), resolution tags dropped before version matching (`RESOLUTION`), `apk` as noise, `old`/`regular`/`unrated`/`patched`/`cracked` as trailing noise. |
| 2026-10-13 | `run_job` ignores a redelivered message for a job that already ended unless it is a Celery retry, so a job marked failed after a worker restart no longer comes back hours later and blocks the queue. |
| 2026-10-13 | Provider notices: `app.providers.notices` keeps short-lived provider warnings in Redis with a TTL; `GET /api/metadata/notices` returns those of the caller's enabled providers (`ProviderNotice`); the topbar `ProviderNotices` icon shows them and toasts new ones. |
| 2026-10-14 | Versions: entries matched to one provider record form one game across file names and episodes, and a file-name group matched to different records splits (`GroupingService._merge_matched`, `version_rank`). Identify changes only the chosen file unless `all_versions` is set. Covers: `fit_cover` handles every raster format and makes animated covers still (`ArtworkService._as_card`); a game without cover art gets one cut from its banner. |
| 2026-10-15 | Grouping also joins matched entries identified as the same name on different providers or records (`_game_keys`: provider record and `title_key` of the title), so one game matched once on VNDB and once on another source becomes one card with versions. |
| 2026-10-15 | Game page actions menu: Hide game / Unhide game marks every version restricted (locked) or clears it (`GameHero` `onToggleHidden`). README rewritten for users with screenshots under `docs/screenshots/`; the design narrative moved to `docs/architecture/Project-Overview.md`. |
