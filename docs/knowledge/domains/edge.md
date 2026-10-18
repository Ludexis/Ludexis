# Domain: Edge

Edge devices are machines (a laptop, another PC) that scan drives attached to them and send the listings to the server, which keeps the only database. A device runs the backend image with `ludexis edge`, has no database or Redis of its own, and talks to the server over HTTPS with a per-device token, polling every 15 seconds (it needs no open port). Libraries get a location: the server (`libraries.edge_id` empty) or one edge device.

## Files
| File | Role | Key symbols |
|---|---|---|
| `backend/app/api/edge_devices.py` | Router `/edge-devices` (web app, `ACCESS_ADMIN`) | `list_edge_devices`, `create_edge_device`, `update_edge_device`, `delete_edge_device`, `rotate_edge_token`, `download_edge_bundle` |
| `backend/app/api/edge_agent.py` | Router `/edge/agent` (edge token only) | `poll`, `libraries`, `library_state`, `job_progress`, `job_failed`, `metadata_result`, `job_report` |
| `backend/app/services/edge.py` | Service | `EdgeService.poll`, `library_state`, `progress`, `fail`, `save_report`, `ingest`, `apply_metadata`, `bundle`, `rotate_token`, `delete` |
| `backend/app/core/edge_auth.py` | Token auth | `new_edge_token`, `hash_edge_token`, `get_current_edge` |
| `backend/app/models/edge_device.py` | Model | `EdgeDevice` (`status` property: PENDING / ONLINE / OFFLINE) |
| `backend/app/schemas/edge.py` | Schemas | `EdgeDeviceRead`, `EdgeDeviceCreated`, `EdgePoll`, `EdgePollResponse`, `EdgeLibraryState`, `EdgeMetadataResult`, ... |
| `backend/app/edge/agent.py` | The device process (`python -m app.edge`) | `EdgeAgent.poll`, `run_forever`, `scan`, `main` |
| `backend/app/edge/drives.py` | Drives the device sees and where a library's folder is | `linux_drives`, `windows_drives`, `library_root`, `drive_key` |
| `backend/app/edge/status.py` | `ludexis-edge status` summary, run inside the device container | `main` |
| `backend/app/edge/bundle/` | Install files served as a zip: `env.template`, `linux/` (compose, `install.sh`, `ludexis-edge`, udev rule, `ludexis-drive@.service`), `windows/` (compose, `install.ps1`, `ludexis-edge.ps1`, `ludexis-edge.cmd`) | |
| `backend/bin/ludexis` | Container entry point: `server`, `worker`, `beat`, `edge`, `health <role>` | |
| `backend/app/tasks/scan_tasks.py` | `ingest_edge_report_task` stores an uploaded listing under the same job | |
| `frontend/app/(app)/admin/edge/page.tsx` | Page `/admin/edge`: devices, status, drives, toggles, add (token and install files shown once) | `DashboardEdge` |
| `frontend/components/admin/library-dialog.tsx` | Library dialog: Name, Folder path, Location (server or an edge device) | `LibraryDialog` |
| `frontend/lib/api/edge.ts` | API module | `edgeApi` |
| `frontend/lib/types/edge.ts` | Types | `EdgeDevice`, `EdgeDeviceCreated`, `EdgeDrive` |
| `backend/tests/test_edge_api.py`, `test_edge_ingest.py`, `test_edge_drives.py` | Backend tests | |

## Graph IDs
| Type | ID |
|---|---|
| Domain | `domain:edge` |
| Router | `router:app.api.edge_devices`, `router:app.api.edge_agent` |
| Service | `service:app.services.edge.EdgeService` |
| Model / Table | `model:app.models.edge_device.EdgeDevice`, `table:edge_devices` |
| Module | `module:app.core.edge_auth` |
| Task | `task:app.tasks.scan_tasks.ingest_edge_report_task` |
| Migration | `migration:c6d1f8a2e940`, `migration:d2f7a9c41b63` |
| Page | `page:/admin/edge` |
| ApiModule / TypeModule | `apimod:lib/api/edge`, `typemod:lib/types/edge` |

## API Endpoints
| Method | Full path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/edge-devices/` | ACCESS_ADMIN | Devices with status, drives, libraries |
| POST | `/api/edge-devices/` | ACCESS_ADMIN | Add a device; returns its token once |
| PATCH | `/api/edge-devices/{edge_id}` | ACCESS_ADMIN | Rename, `autoscan`, `match_metadata` |
| DELETE | `/api/edge-devices/{edge_id}` | ACCESS_ADMIN | Revoke; its libraries fall back to the server and its waiting jobs are cancelled |
| POST | `/api/edge-devices/{edge_id}/token` | ACCESS_ADMIN | New token; the old one stops working |
| POST | `/api/edge-devices/{edge_id}/bundle` | ACCESS_ADMIN + current token | Zip with compose, `.env`, installer for `linux` or `windows` |
| POST | `/api/edge/agent/poll` | edge token | Check-in: version, host, OS, drives; returns settings, libraries, waiting jobs, metadata requests |
| GET | `/api/edge/agent/libraries` | edge token | Its libraries (used by `ludexis-edge restart`) |
| GET | `/api/edge/agent/libraries/{library_id}/state` | edge token | Size, time and hash presence per relative path |
| POST | `/api/edge/agent/jobs/{job_id}/progress` | edge token | Progress; answers `cancelled` |
| POST | `/api/edge/agent/jobs/{job_id}/fail` | edge token | The device could not run the job |
| POST | `/api/edge/agent/jobs/{job_id}/report` | edge token | Gzipped listing of the whole library |
| POST | `/api/edge/agent/metadata` | edge token | A metadata match the device looked up |

## Tables
`edge_devices`; `libraries.edge_id`, `drive_id`, `drive_subpath`, `drive_mount`, `pending_changes` (unique index `uq_libraries_location_path` on `(coalesce(edge_id, ''), path)`); `job_history.edge_id`, `library_id`.

## Change guide
- New agent call: route in `backend/app/api/edge_agent.py`, logic in `EdgeService`, the call in `backend/app/edge/agent.py`, a test in `backend/tests/test_edge_api.py`.
- Install files: `backend/app/edge/bundle/`; Linux files keep LF and Windows files CRLF (`backend/.gitattributes`). `EdgeService.bundle` fills `env.template` with the server URL and token.
- Drive identity: `drive_key` compares the last 8 hex digits, which is what Linux (NTFS UUID, exFAT serial) and Windows (32-bit volume serial) have in common.

## Notes
- The server never sees an edge drive. A device sends the full listing of a library (paths, sizes, times, hashes it computed); `ScannerService.ingest_report` stores it through the same ingest code as a server scan. A path counts as gone only when it lies inside the reported library and is missing from the listing.
- An empty listing for a library with entries, or a reported error (drive not connected, path not found, letter not mapped), sets the library OFFLINE and changes nothing else.
- Guards keep the server's own behaviour unchanged: the server scan and `verify_archives` skip edge libraries, `JobService.active_job` ignores edge jobs, and `fail_stale_jobs` leaves edge jobs alone until a PENDING one is 7 days old or the device of a RUNNING one has not checked in for an hour.
- Autoscan (default on): a poll that reports a drive which was not there at the previous poll queues an `INCREMENTAL_SCAN` for the libraries on it, and for offline libraries whose drive is not known yet.
- Automatic scans (job without `user_id`: autoscan, schedules) hash nothing: files matching by relative path and size are kept with their stored hash, removed files become MISSING, and new or resized files are sent as `pending` and counted in `libraries.pending_changes`. The Libraries page then asks to check them; a scan a user starts hashes only new or changed files and resets the count. A drive that comes back is never re-hashed in full.
- With `match_metadata` on, the poll hands out up to 20 never-tried entries; the device runs `MetadataService.auto_match` and `MetadataService.get_merged_details` and posts the result, which `EdgeService.apply_metadata` stores with the server's thresholds. The server queues no enrichment for those entries.
- Flow: `docs/knowledge/flows/edge-scan.md`.
