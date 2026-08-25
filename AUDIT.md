# Production Readiness Audit — 2026-08-25

## Summary

**Ready for its intended deployment** (single-user, loopback-bound desktop/web
app), with fixes recommended before any multi-user or network-exposed reuse.
Backend fundamentals are strong: 423 passing tests, real coverage on the
critical paths (encryption round-trip, backup/restore integrity, OAuth token
refresh), comprehensive health checks, graceful shutdown, WAL-safe SQLite with
boot-time recovery. The genuine gaps are: near-zero frontend test coverage
(3 test files vs ~68 source files), a handful of fire-and-forget async tasks
that fail silently, and upload endpoints that read the whole file into memory
before the size check. No secrets are in git (verified: `backend/.env` is
gitignored and absent from all history; the OAuth secret it contains was never
committed).

## Blockers (must fix before prod)

- [ ] `backend/app/routers/media.py:36` and `routers/recordings.py:29` —
  `await file.read()` loads the entire upload into memory **before** the 25 MB
  check in `MediaService.upload()` (`services/media_service.py:96`). A local
  user pasting a multi-GB file OOMs the process mid-transaction. Fix: check
  `Content-Length`/stream to disk in chunks before validating.
- [ ] `backend/app/routers/ai.py:282` — fire-and-forget
  `asyncio.create_task(_pull())` for Ollama model pulls: no error callback, no
  task registry (shutdown in `main.py:140` only cancels enrichment tasks).
  A failed pull leaves the UI stuck on "pulling" forever with nothing logged.
  Same pattern: `routers/tts.py:282` (prewarm), `services/enrichment_service.py:25`.
  Fix: `task.add_done_callback(...)` that logs, and register tasks for shutdown
  cancellation.

## High priority

- [ ] Frontend test coverage — `EntryEditor.vue` (1,519 lines) and
  `NoteEditor.vue` (1,641 lines) are the app's core surfaces with zero tests;
  all 9 stores and 18 of 19 composables untested (only `useFormat` has a spec).
  CI runs what exists and gates on it (`.github/workflows/ci.yml:35,66`), so
  the gap is coverage, not gating.
- [ ] `backend/app/core/config.py:217` — `APP_ENV` defaults to `development`;
  `validate_production()` (config.py:341) only runs when explicitly set to
  `production`, so an accidental prod-style deploy silently skips the
  SECRET_KEY/loopback checks. Fail-safe beats opt-in.
- [ ] `backend/app/core/config.py:224` — `SECRET_KEY` default
  `change-me-before-production` guards all encrypted OAuth credentials when
  running bare (`python -m app.main`); the managed launcher path generates a
  real key, the bare path does not.

## Medium / Low

- [ ] medium `services/scheduler_service.py:479` — schedule-restore failures
  warn and continue: auto-backup can silently not be armed (partially
  mitigated — `startup_checks.check_backup_health` self-heals on next boot,
  `main.py:112`).
- [ ] medium `services/media_service.py:179` — `upload_from_path` reads the
  whole file into memory (2 GiB import cap, `config.py:237`); same chunked-
  read fix as the blockers covers it.
- [ ] medium `services/reminder_service.py:94` — `notify-send` argv includes
  DB-sourced title/message; argv-list (no shell) and localhost-only make this
  low exploitability, but a leading `-` in a reminder title becomes a flag.
- [ ] low `routers/{box,onedrive,dropbox,google_drive}.py` — OAuth
  `REDIRECT_URI` hardcoded to port 18765 while `--port` is configurable
  (`main.py:528`): running on any other port breaks all four cloud-connect
  flows.
- [ ] low `frontend/package.json:4` — version `0.0.0` placeholder out of sync
  with the 4 checked sources (`scripts/check_version.py`); runtime version
  comes from `VITE_APP_VERSION` so it's cosmetic.
- [ ] low `scripts/build-web-deb.sh:234` — single-instance PID check doesn't
  verify the PID still belongs to LifeLogr (PID-recycle false positive).
- [ ] low Docker path: no root `.env.example`; `docker-compose.yml` is
  loopback-bound and sane, but discoverability of required vars is poor.
- [ ] accepted `--no-access-log` in the packaged launcher — deliberate noise
  reduction for a local app.

## Explicitly out of scope / accepted risk

- **No API authn/authz** — by design (ADR `docs/04-design/ADR/002-no-auth-
  local-endpoints.md`): loopback binding + scoped CORS is the threat-model
  boundary. Revisit before any non-loopback deployment.
- **Real Google OAuth secret in untracked `backend/.env`** — never committed
  (verified across all refs); rotation still recommended as hygiene since the
  file sat on disk since May, but it is not a repo leak.
- **Desktop (Rust) side has `cargo check` in CI only** — the Tauri shell is
  thin (sidecar + window); accepted.
- **Emoji in OAuth success pages** — established human tone, kept.

## Verification status

- Test baseline: 423 backend + 21 frontend passing (Phase 0, re-verified
  after Phases 1–3).
- Claims in this audit were spot-checked in the main thread: `.env` git status,
  upload size-check location, CI test invocation, REDIRECT_URI hardcoding,
  `validate_production()` gating — all confirmed as stated above.
