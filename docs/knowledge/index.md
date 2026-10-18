# Ludexis Knowledge Graph

The verified map of the Ludexis codebase: which files implement which feature, how a request moves from a page to the database, and what depends on what.
Use it before searching the repo. It is checked against the code by `queries/validate.py`.

---

## How to Use It

| I want to... | Go to |
|--------------|-------|
| Find every file for a feature (backend and frontend) | `domains/<domain>.md`, or `python docs/knowledge/queries/kg.py files <domain>` |
| Follow a request end to end | `flows/<flow>.md`, or `kg.py trace <page or route id>` |
| See what breaks if I change something | `kg.py impact <id>` (includes the tests that cover it) |
| Look up a class, route, page, or function | `kg.py find <text>`, then `kg.py show <id>` |
| Understand the ID formats and edge types | `_schema.md` |
| Know why the graph is modeled this way | `_decisions.md` |
| Know what is covered, known gaps, and open bugs | `_state.md` |
| Check the graph is still accurate | `python docs/knowledge/queries/validate.py` |

---

## Domains

Each domain file lists its backend and frontend files, graph IDs, endpoints (with permissions and frontend callers), tables, and a change guide.

| Domain | Backend router | Frontend pages |
|--------|----------------|----------------|
| [admin](domains/admin.md) | `backend/app/api/admin.py` | `/admin`, `/admin/analytics`, `/admin/monitoring`, `/admin/settings` |
| [archive](domains/archive.md) | `backend/app/api/archive_entries.py` | `/`, `/library`, `/archive/[id]`, `/admin/duplicates` |
| [artwork](domains/artwork.md) | `backend/app/api/artwork.py` | `/admin/artwork` |
| [audit](domains/audit.md) | (used by most routers through `AuditService` / `AuditLogService`) | `/admin/audit-logs` |
| [auth](domains/auth.md) | `backend/app/api/auth.py`, `backend/app/api/setup.py` | `/auth/login`, `/auth/setup`, `/account` |
| [collection](domains/collection.md) | `backend/app/api/collections.py` | `/collections`, `/collections/[id]`, `/collections/[id]/edit`, `/collections/new`, `/admin/collections` |
| [core](domains/core.md) | `backend/app/api/health.py`, `backend/main.py`, `backend/app/core/` | root layout, shared UI |
| [edge](domains/edge.md) | `backend/app/api/edge_devices.py`, `backend/app/api/edge_agent.py` | `/admin/edge` (and the Location field of `/admin/libraries`) |
| [developer](domains/developer.md) | `backend/app/api/developers.py` | `/developers`, `/developers/[id]` |
| [franchise](domains/franchise.md) | `backend/app/api/franchises.py` | `/franchises`, `/franchises/[id]` |
| [genre](domains/genre.md) | (no router; genres are synced by metadata enrichment) | none |
| [job](domains/job.md) | `backend/app/api/jobs.py`, `backend/app/api/job_monitor.py` | `/admin/jobs` |
| [library](domains/library.md) | `backend/app/api/libraries.py` | `/admin/library` |
| [metadata](domains/metadata.md) | `backend/app/api/metadata.py` | `/admin/metadata` |
| [publisher](domains/publisher.md) | `backend/app/api/publishers.py` | `/publishers`, `/publishers/[id]` |
| [rbac](domains/rbac.md) | `backend/app/api/users.py`, `roles.py`, `permissions.py` | `/admin/users`, `/admin/permissions` |
| [scan](domains/scan.md) | `backend/app/api/scan.py` | (started from admin pages) |
| [search](domains/search.md) | `backend/app/api/search.py` | `/search` |
| [storage](domains/storage.md) | `GET /media/{path:path}` in `backend/main.py` | (artwork URLs via `frontend/lib/media.ts`) |
| [tag](domains/tag.md) | `backend/app/api/tags.py` | `/tags`, `/tags/[id]` |

## Flows

| Flow | Covers |
|------|--------|
| [frontend-request](flows/frontend-request.md) | Page -> `lib/api/<x>.ts` -> `lib/api/client.ts` (base URL, auth header, refresh) -> router -> service -> repository -> DB |
| [auth](flows/auth.md) | Login, token refresh, logout, first-run setup |
| [rbac](flows/rbac.md) | Users, roles, permissions, permission checks on routes |
| [scan](flows/scan.md) | Full and incremental library scans, duplicate detection |
| [metadata](flows/metadata.md) | Provider search, auto-match, scheduled refresh, conflict resolution |
| [artwork](flows/artwork.md) | Upload, replace, auto-download, validation, garbage collection |
| [edge-scan](flows/edge-scan.md) | Library on an edge device -> check-in -> scan and hash on the device -> listing uploaded -> stored by the server worker |
| [jobs](flows/jobs.md) | Job start, Celery dispatch, progress, cancel, beat schedule |

---

## Repository at a Glance

```
Ludexis/
├── backend/                     FastAPI 0.143 + SQLAlchemy 2.0 + Celery 5.4 + PostgreSQL + Redis
│   ├── main.py                  app factory, CORS, /media, /healthz, /api/metrics
│   ├── app/api/                 22 routers, 100 routes under /api (+3 in main.py)
│   ├── app/services/            22 services, bootstrap (first-start admin) (+ ArchiveScanItem helper class)
│   ├── app/repositories/        15 repositories + BaseRepository
│   ├── app/models/              18 models + association_tables.py (27 tables)
│   ├── app/schemas/             22 modules, 79 Pydantic schemas
│   ├── app/tasks/               celery_app + scan/metadata/artwork/scheduler tasks (7 tasks)
│   ├── app/providers/           MetadataProvider base, VNDB, IGDB (+client), Steam, Manual
│   ├── app/core/ app/utils/ app/db/   config, auth/permissions, security, logging, metrics, enums, helpers
│   ├── alembic/versions/        12 migrations from the baseline ecabaf1d5dab
│   ├── scripts/                 seed_demo, reset_password
│   └── tests/                   36 pytest files
├── frontend/                    Next.js 16.3 + React 19 + TypeScript 5.7 + Tailwind 4
│   ├── app/                     24 pages; root, (app), (auth) and admin layouts
│   ├── components/              77 components in admin, brand, browse, common, game, library, media, shell, ui
│   ├── hooks/ contexts/         use-api, use-protected-route, use-review-count; auth-context
│   ├── lib/api/                 21 API modules, 89 API functions (+ client.ts)
│   ├── lib/types/               20 type modules, 63 exported types
│   └── e2e/                     7 Playwright specs (+ 35 vitest files beside the code)
└── docs/knowledge/              this graph
```

## Directory Layout

```
docs/knowledge/
├── index.md          this file
├── _schema.md        entity types, ID formats, relationship types
├── _decisions.md     modeling decisions (ADRs)
├── _state.md         coverage, known gaps, findings, maintenance
├── domains/          19 per-feature files
├── flows/            7 end-to-end flows
├── queries/          validate.py, kg.py (stdlib only)
└── graph/
    ├── entities/       backend_*.jsonl, frontend_*.jsonl
    └── relationships/  backend_*.jsonl, frontend_*.jsonl
```

### Graph Files

Load only the file you need.

| File | Contents |
|------|----------|
| `entities/backend_api_routes.jsonl` | Routers and APIRoutes (method, path, `full_path`, permissions, schemas) |
| `entities/backend_services.jsonl` / `backend_repositories.jsonl` / `backend_models.jsonl` / `backend_schemas.jsonl` | Classes with public methods and line ranges |
| `entities/backend_tasks.jsonl` / `backend_providers.jsonl` | Celery tasks (schedules), metadata providers, external providers |
| `entities/backend_db_tables.jsonl` / `backend_db_enums.jsonl` / `backend_migrations.jsonl` | Tables (columns, FKs, indexes), enums, migrations |
| `entities/backend_modules.jsonl` | `app/core`, `app/utils`, `app/db`, `main.py`, seed scripts |
| `entities/backend_test_files.jsonl`, `frontend_tests.jsonl` | Test files with test counts |
| `entities/backend_domains.jsonl` / `backend_layers.jsonl` | The 19 domains and 16 layers |
| `entities/frontend_pages.jsonl` / `frontend_components.jsonl` / `frontend_hooks_contexts.jsonl` | Pages (auth guard, API calls), components, hooks, context |
| `entities/frontend_api.jsonl` | API modules and functions (HTTP method, path, backend route) |
| `entities/frontend_types.jsonl` / `frontend_lib.jsonl` | TS types, lib utilities |
| `relationships/backend_api_to_service.jsonl` | Route -> service/repository (`CALLS`), schemas (`ACCEPTS`/`RETURNS`), domains |
| `relationships/backend_service_deps.jsonl` / `backend_repo_deps.jsonl` / `backend_task_deps.jsonl` / `backend_provider_deps.jsonl` | Dependencies and table access |
| `relationships/backend_model_deps.jsonl` / `backend_schema_deps.jsonl` / `backend_migration_deps.jsonl` | ORM relations, schema mapping, migration tables |
| `relationships/backend_module_deps.jsonl` | Imports of core/utils/db modules |
| `relationships/backend_cross_layer.jsonl` | Derived route -> repository/table flows |
| `relationships/frontend_ui_deps.jsonl` | Renders, hooks, context, API calls, imports |
| `relationships/frontend_api_deps.jsonl` | API function -> backend route (`CALLS_ENDPOINT`), contains, imports |
| `relationships/frontend_type_deps.jsonl` | TS type -> backend schema (`MIRRORS`) |
| `relationships/*_test_coverage.jsonl` | Test file -> covered entity |

## Graph Statistics

Verified at commit `4d798fc` on 2026-10-06. Regenerate with `validate.py --stats-json`.

| | Count |
|-|-------|
| Entities | 776 |
| Relationships | 3,598 |

| Entity type | Count | Entity type | Count |
|-------------|-------|-------------|-------|
| APIRoute | 99 | Component | 57 |
| Schema | 74 | ApiFunction | 110 |
| DBTable | 29 | TestFile | 83 (32 backend, 51 frontend) |
| Router | 22 | Type | 69 |
| Service | 22 | Page | 33 |
| Domain | 19 | ApiModule | 22 |
| Model | 20 | TypeModule | 21 |
| Module | 19 | LibUtil | 10 |
| Repository | 16 | Hook | 1 |
| Layer | 16 | Context | 1 |
| DBEnum | 7 | Layout | 1 |
| Task | 7 | Class | 1 |
| Provider | 6 | Migration | 7 |
| ExternalProvider | 4 | | |

| Relationship type | Count | Relationship type | Count |
|-------------------|-------|-------------------|-------|
| BELONGS_TO | 1308 | DEPENDS_ON | 40 |
| IMPORTS | 396 | ACCEPTS | 27 |
| RENDERS | 294 | ACCESSES | 31 |
| TESTS | 206 | CREATES | 29 |
| CONTAINS | 156 | MAPS_TO | 27 |
| CALLS | 144 | USES_CLIENT | 20 |
| CALLS_API | 138 | DEFINES | 27 |
| FLOWS_TO | 105 | MANY_TO_MANY | 15 |
| CALLS_ENDPOINT | 115 | ASSOCIATED_THROUGH | 13 |
| EXPOSES | 99 | MANY_TO_ONE | 12 |
| RETURNS | 68 | ONE_TO_MANY | 11 |
| READS | 63 | USES_PROVIDER | 7 |
| WRITES | 56 | CONSUMES | 12 |
| EXTENDS | 45 | NESTS | 9 |
| MIRRORS | 42 | DISPATCHES | 8 |
| USES_CONTEXT | 37 | USES_ENUM | 5 |
| USES_HOOK | 31 | PROVIDES | 2 |

---

## Keeping It Accurate

1. After changing code, update the affected records (stable IDs make this a patch, not a rebuild) and set `last_verified_at` and `commit_hash`.
2. Run `python docs/knowledge/queries/validate.py`. It must exit 0 before the change is committed. A staleness WARN means code has changed since the recorded `commit_hash`.
3. If types, ID formats, or edge types change, update `_schema.md`, `_decisions.md`, and `validate.py` together.
4. Update the counts above from `validate.py --stats-json` and the status in `_state.md`.

Related project docs: `docs/architecture/` (architecture, backend, data model, pipeline), `docs/api/API-Guide.md`, `docs/deployment/Deployment.md`.
