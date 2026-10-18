# Flow: Edge scan

From adding a library on an edge device to its entries on the server. Domain doc: `docs/knowledge/domains/edge.md`.

## 1. Library and job
```
frontend/components/admin/library-dialog.tsx -> librariesApi.create({name, path, edge_id})
  POST /api/libraries/  ACCESS_ADMIN -> LibraryService.create (path unique per machine)
  edge_id set -> JobService.queue_edge_scan(db, user, INCREMENTAL_SCAN, library)
       job_history row PENDING with edge_id and library_id; no Celery message
Scan button / scan of everything / scheduled scan -> JobService.start_job
  library on an edge -> queue_edge_scan (one waiting or running scan per library)
  no library -> server Celery job, plus queue_edge_scan for every enabled edge library
```

## 2. Check-in (every 15 s)
```
backend/app/edge/agent.py EdgeAgent.poll
  drives = visible_drives(os)      backend/app/edge/drives.py (Linux: mountinfo + udev; Windows: /state/drives.json)
  POST /api/edge/agent/poll -> EdgeService.poll
       last_seen_at, version, hostname, os, drives
       autoscan: a newly connected drive -> queue_edge_scan for its libraries
       <- libraries, PENDING/RUNNING jobs without task_id, metadata requests
  writes /state/drives (drive ids of its libraries) for the Linux mount helper
```

## 3. Scan on the device (worker thread; check-ins go on)
```
EdgeAgent.scan(job_id, library)
  library_root(os, library, drives) -> folder, drive, subpath   (by drive_id when known, else the typed path)
  missing -> report {error} (library goes OFFLINE)
  ScannerService._discover_items(root) ; _relative_path
  GET /api/edge/agent/libraries/{library_id}/state -> hash only new or changed files (all when verify)
  POST /api/edge/agent/jobs/{job_id}/progress every 2 s (cancelled -> stop)
  POST /api/edge/agent/jobs/{job_id}/report  (gzip JSON: library_id, drive, items)
```

## 4. Storing the report (server worker)
```
EdgeService.save_report -> /data/edge-reports/<job>.json.gz ; ingest_edge_report_task.apply_async ; job.task_id
ingest_edge_report_task -> run_job -> EdgeService.ingest -> ScannerService.ingest_report
  drive id, subpath, mount stored on the library ; OFFLINE on error or an empty listing
  exists(entry) answered from the listing ; hash_files False
  _ingest_batches (create, update, move by relative_path or hash, MISSING marking, reconnect verify, regroup)
  -> _queue_enrichment (skipped when the device matches metadata itself)
```

## Entities
`router:app.api.edge_agent`, `router:app.api.edge_devices`, `service:app.services.edge.EdgeService`, `service:app.services.scanner.ScannerService`, `service:app.services.job.JobService`, `task:app.tasks.scan_tasks.ingest_edge_report_task`, `table:edge_devices`, `table:libraries`, `table:job_history`, `page:/admin/edge`, `apimod:lib/api/edge`.
