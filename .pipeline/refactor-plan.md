# Phase 2 Refactor Plan (dry run) — 2026-08-25

Units in apply order. No behavior change in any unit. Scouts' findings that
are *style* (error-format consolidation) are deferred to Phase 3; findings that
are *correctness* (silent catches) are deferred to Phase 4 (audit) — Phase 2
only removes bloat.

## Unit A — OAuth decrypt dedup (backend)

`json.loads(decrypt(...))` + warning-log repeated 8× across 4 routers
(onedrive.py:48-54, dropbox.py:48-53,85-89, google_drive.py:48-54,102-108,
box.py:53-58,98-103). `_oauth_helpers.py` already exists as the shared home
for exactly this plumbing (its docstring says so).

Action: add `load_stored_credentials(config) -> dict` to `_oauth_helpers.py`
(warns + returns {} on failure — same observable behavior), replace all 8
sites. Removes ~30 lines, one canonical decrypt path.

## Unit B — dead frontend exports

- `openExternal` (utils/externalLink.ts:28-42) — zero callers. The click
  interceptor below it handles all dispatch. Remove function (keep the
  doc block context by trimming it into the interceptor's comment where
  relevant).
- `resolveTagIds` (utils/tags.ts:22-40) — zero callers; EntryEditor resolves
  tags through the store directly. Remove.

Removes ~35 lines. `extractHashtags` and `isExternalHref` stay (both used,
verified).

## NOT doing (with reasons — logged for AUDIT.md)

- **Inline usePasteMedia/useTauriDragDrop composables** — each has 2 real
  consumers (EntryEditor + NoteEditor); that's a legitimate shared abstraction,
  not a wrapper with one caller. Inlining would *duplicate* code. Skip.
- **Consolidate `e instanceof Error ? e.message : String(e)` (50 sites)** —
  that's a style/consistency concern; scheduled for Phase 3 decision (extract
  `errMsg()` helper vs leave as the codebase's established idiom).
- **URLSearchParams builders** — 4 occurrences with genuinely different param
  shapes; a shared builder adds a generic wrapper for ~10 net lines saved.
  Boring-over-clever says leave it.
- **Startup `except Exception` warn-only blocks (main.py ×10)** — justified
  boundary handlers per scout; each guards an independent startup concern so
  one failed check can't kill the app. Keep.
- **scheduler_service bare excepts ×10** — background-job resilience; flagged
  for Phase 4 audit review (swallow-vs-log), not Phase 2 removal.
- **settings_service double try/except** — file I/O at a real boundary (user's
  settings file can genuinely be corrupt); defensive but defensible. Audit
  note, no change.
