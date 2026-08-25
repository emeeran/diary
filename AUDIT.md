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

## Missed by pipeline, caught by blind review

(An independent, context-blind reviewer — see `.pipeline/blind-review.md` —
found these after Phases 1–4; they are genuine pipeline misses, recorded here
rather than quietly patched.)

- [ ] high `routers/media.py:65` — the **batch** upload endpoint repeats the
  whole-file `file.read()` in a loop, compounding the OOM blocker. The audit
  had caught the single-upload and recordings paths; it missed the batch one.
- [ ] medium `services/media_service.py:158-186` vs
  `services/note_media_service.py:141-164` — the from-path sandbox exists as
  two ~25-line copy-pasted implementations drifting independently. This is
  exactly the duplication Phase 2 was supposed to find, and didn't. One
  shared, unit-tested helper is the fix.
- [ ] medium the from-path sandbox blocklist misses `~/.aws`, `~/.kube`,
  `~/.netrc`, `~/.docker`, `~/.var/app` (Flatpak) — blocklist-vs-moving-target
  problem. Consider inverting to an explicit allowlist of media file
  extensions + size cap before `read_bytes()`.
- [ ] low `config.py:16` — `_default_data_dir` docstring references dead
  `DIARIUM_DATA_DIR` env var; `.gitignore` still lists `*.diary`. Naming
  residue from the app's pre-LifeLogr identity.
- [ ] low `main.py:201-242` — rate limiter never fires in the supported
  deployment (disabled unless production env, pointless on loopback). ~40
  dead lines; either delete or document as server-only.
- [ ] low stale `.pyc` files for ~25 deleted modules (email_service,
  contacts_sync, planner_service…) sit in `__pycache__` on disk — untracked,
  but `make clean` should sweep them.

Both agree on (higher confidence): the OOM upload pattern as a blocker,
fire-and-forget task handling, frontend test gap, SECRET_KEY/APP_ENV default
risk, OAuth-secret rotation as hygiene.

Disagreement (surfaced, not resolved): the blind reviewer asked "where is
`validate_production()` even called?" — verified answer: `database.py:366`,
gated on `is_production and not sidecar`. The reviewer missed the call site
but the substance (opt-in gating) stands; the High finding above is correct
as written.

## Verification status

- Test baseline: 423 backend + 21 frontend passing (Phase 0, re-verified
  after Phases 1–3).
- Claims in this audit were spot-checked in the main thread: `.env` git status,
  upload size-check location, CI test invocation, REDIRECT_URI hardcoding,
  `validate_production()` gating — all confirmed as stated above.
- The blind reviewer independently ran the backend suite: 423 passed,
  matching.
