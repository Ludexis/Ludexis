# Domain: Job

Background job records (`job_history`) and their Celery tasks: start, cancel, list, and live worker/queue monitoring. Every `JobType` maps to a real task.

## Files
| File | Role | Key symbols |
|---|---|---|
| `backend/app/api/jobs.py` | Router `/jobs` | `start_job`, `cancel_job`, `list_jobs`, `read_job` |
| `backend/app/api/job_monitor.py` | Router `/job-monitor` | `list_workers`, `queue_stats` |
| `backend/app/services/job.py` | Service | `JobService.list_jobs`, `get_job`, `start_job`, `queue_edge_scan`, `active_job`, `_select_task`, `cancel_job`, `is_cancelled` |
| `backend/app/tasks/job_runner.py` | Shared task lifecycle | `run_job`, `summarize`, `JOB_TASK_OPTIONS` |
| `backend/app/services/job_monitor.py` | Service (Celery inspect) | `JobMonitorService.workers`, `active`, `reserved`, `stats` |
| `backend/app/repositories/job_history.py` | Repository | `JobHistoryRepository.list_items`, `count_by_status`, `list_by_status`, `get_by_task_id` |
| `backend/app/models/job_history.py` | Model | `JobHistory` |
| `backend/app/schemas/job_history.py` | Schemas | `JobHistoryCreate`, `JobHistoryRead`, `JobHistoryBase`, `JobStartRequest` |
| `backend/app/tasks/celery_app.py` | Celery app, beat schedule | `celery_app` |
| `backend/app/tasks/scan_tasks.py` | Scanner-backed job tasks | `verify_integrity_task`, `detect_duplicates_task` (plus the scan tasks, see scan domain) |
| `backend/app/utils/enums.py` | Enums | `JobType`, `JobStatus` |
| `backend/app/core/config.py` | Settings | `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND`, `JOB_MAX_RETRIES`, `JOB_RETRY_BACKOFF_MAX` |
| `frontend/app/(app)/admin/page.tsx` | Admin dashboard quick actions | `jobsApi.start`, `scansApi.runFull`, `adminApi.getRecentJobs` |
| `frontend/lib/api/jobs.ts` | API module | `jobsApi.getAll`, `getById`, `getPending`, `getRunning`, `getCompleted`, `start`, `cancel` |
| `frontend/lib/api/job-monitor.ts` | API module | `jobMonitorApi.getStats`, `getWorkers` |
| `frontend/lib/types/jobs.ts` | Types | `JobType`, `JobStatus`, `JobHistory`, `JobHistoryCreate`, `JobMonitorStats`, `JobMonitorWorker`, `ScanStatus` |
| `backend/tests/test_jobs.py`, `backend/tests/test_scan_jobs_api.py` | Backend tests | |
|    `frontend/e2e/jobs.spec.ts` | Frontend tests | |
| `frontend/components/admin/scheduled-tasks.tsx` | Component | `ScheduledTasks` |
| `frontend/components/shell/job-labels.ts` | Component | `jobLabel` |
| `frontend/components/shell/jobs-indicator.tsx` | Component | `announceJobsChanged`, `JobsIndicator` |
| `frontend/app/(app)/admin/tasks/page.tsx` | Page `/admin/tasks` | `DashboardTasks` |

## Graph IDs
| Type | ID |
|---|---|
| Router | `router:app.api.jobs`, `router:app.api.job_monitor` |
| Service | `service:app.services.job.JobService`, `service:app.services.job_monitor.JobMonitorService` |
| Repository | `repo:app.repositories.job_history.JobHistoryRepository` |
| Model | `model:app.models.job_history.JobHistory` |
| Schema | `schema:app.schemas.job_history.JobHistoryCreate`, `schema:app.schemas.job_history.JobHistoryRead` |
| Task | `task:app.tasks.scan_tasks.verify_integrity_task`, `task:app.tasks.scan_tasks.detect_duplicates_task` (scan domain; dispatched by `JobService`) |
| DBEnum | `dbenum:job_type`, `dbenum:job_status` |
| ExternalProvider | `extprov:Celery` |
| Table | `table:job_history` |
| Page | `page:/admin`, `page:/admin/tasks` |
| ApiModule | `apimod:lib/api/jobs`, `apimod:lib/api/job-monitor` |
| TypeModule | `typemod:lib/types/jobs` |
| TestFile | `test:backend/tests/test_jobs.py`, `test:backend/tests/test_scan_jobs_api.py`, `test:frontend/e2e/jobs.spec.ts` |
| Component | `comp:components/admin/scheduled-tasks`, `comp:components/shell/job-labels`, `comp:components/shell/jobs-indicator` |

## API Endpoints
| Method | Full path | Handler | Permission | Frontend caller |
|---|---|---|---|---|
| POST | `/api/jobs/start` | `start_job` (`backend/app/api/jobs.py`) | RUN_SCANS | `apifn:lib/api/jobs.jobsApi.start` |
| POST | `/api/jobs/{job_id}/cancel` | `cancel_job` (`backend/app/api/jobs.py`) | RUN_SCANS | `apifn:lib/api/jobs.jobsApi.cancel` |
| GET | `/api/jobs/` | `list_jobs` (`backend/app/api/jobs.py`) | authenticated | `apifn:lib/api/jobs.jobsApi.getAll` |
| GET | `/api/jobs/{job_id}` | `read_job` (`backend/app/api/jobs.py`) | authenticated |  |
| GET | `/api/job-monitor/workers` | `list_workers` (`backend/app/api/job_monitor.py`) | ACCESS_ADMIN | `apifn:lib/api/job-monitor.jobMonitorApi.getWorkers` |
| GET | `/api/job-monitor/stats` | `queue_stats` (`backend/app/api/job_monitor.py`) | ACCESS_ADMIN | `apifn:lib/api/job-monitor.jobMonitorApi.getStats` |

## Tables
`job_history` (insert by `JobService.start_job`; status, progress, retry_count and result written by `run_job`), `audit_logs` (`START_JOB`, `CANCEL_JOB`).

## Change guide
- Make a job type run real work: add a Celery task decorated with `@celery_app.task(**JOB_TASK_OPTIONS)` that returns `run_job(self, job_history_id, label, work)` (pattern: `backend/app/tasks/scan_tasks.py`), import it in `backend/app/services/job.py` and add it to the `JobService._select_task` mapping; make sure `backend/app/tasks/celery_app.py` imports the task module.
- New job type: add to `JobType` in `backend/app/utils/enums.py`, a migration altering the `job_type` PostgreSQL enum in `backend/alembic/versions/`, `JobType` in `frontend/lib/types/jobs.ts`, and `JOB_TYPES` in `frontend/components/admin/jobs-panel.tsx`.
- New `JobHistory` field: `backend/app/models/job_history.py`, `JobHistoryBase` in `backend/app/schemas/job_history.py`, migration, `JobHistory` in `frontend/lib/types/jobs.ts`.
- New monitor metric: `JobMonitorService` (`backend/app/services/job_monitor.py`), route in `backend/app/api/job_monitor.py`, `jobMonitorApi` + `JobMonitorStats`.

## Notes
- `JobService._select_task`: `LIBRARY_SCAN` -> `scan_full_task`, `INCREMENTAL_SCAN` -> `scan_incremental_task`, `METADATA_REFRESH` -> `refresh_metadata_task`, `ARTWORK_REFRESH` -> `validate_artwork_task`, `DUPLICATE_DETECTION` -> `detect_duplicates_task`, `INTEGRITY_VERIFICATION` -> `verify_integrity_task` (a dict lookup; an unmapped type raises `KeyError`).
- `JobService.cancel_job` revokes the Celery task (`terminate=True`) and sets `CANCELED`; jobs not `PENDING`/`RUNNING` are returned unchanged. Scans and metadata refresh also poll the row for `CANCELED` between items.
- `JobService.is_cancelled`, `JobHistoryRepository.list_by_status` and `get_by_task_id` have no callers.
- `JOB_TASK_OPTIONS`: `autoretry_for=(Exception,)`, `retry_backoff=True`, `max_retries=JOB_MAX_RETRIES` (5), `retry_backoff_max=JOB_RETRY_BACKOFF_MAX` (300 s). `run_job` stores `task.request.retries` in `retry_count`, which `frontend/components/admin/jobs-panel.tsx` shows as "retry N".
- `start_job(db, user, job_type, details=None, task_kwargs=None)` passes `task_kwargs` to the task (`entry_ids` for enrichment). `active_job` returns a PENDING/RUNNING server job of a type; the scheduled tasks use it to avoid overlapping runs. A scan of a library on an edge device becomes a PENDING row with `edge_id` and no Celery message (`queue_edge_scan`, one per library); the device runs it and `ingest_edge_report_task` stores its listing (`docs/knowledge/flows/edge-scan.md`). `fail_stale_jobs` leaves edge jobs to their own limits (7 days PENDING, 1 hour without a check-in while RUNNING).
- No stale-job recovery: a job left RUNNING by a killed worker stays RUNNING.
- `GET /api/jobs/` is ordered by `started_at` desc and accepts `job_type`, `status`, `offset`, `limit`.
- `frontend/components/admin/jobs-panel.tsx` (Admin > Scheduled Tasks, Jobs tab) polls every 5 s while any job is active.
- `/api/job-monitor/*` return raw dicts (no `response_model`).
