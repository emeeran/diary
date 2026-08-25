# Phase 1 Purge Plan (dry run) — 2026-08-25

Verified against: md5 dupes, import scans (backend `app/`+`tests/`, frontend
`src/` incl. dynamic-`component :is` check), gitignore coverage, last-commit
dates. `local/` is already untracked (gitignored) — not this pipeline's concern.

## Move to trash2review (high confidence)

| path | reason | confidence | inbound refs |
|---|---|---|---|
| `backend/scripts/export_conv.py` | one-off transcript exporter; hardcoded personal paths to a *previous repo location* (`/home/em/code/wip/diary`); nothing references it | high | 0 |
| `backend/app/schemas/revision.py` | schemas for entry-revision history; `entry_revisions` table is explicitly DROPped at boot as legacy with "no ORM model, no active code path" (`app/core/database.py:504`) | high | 0 |
| `media/Garden.mp3` | byte-identical (md5 `437888e3…`) to `frontend/public/Garden.mp3`, which is the copy `routers/memorial.py` resolves; root copy unreferenced; 1.3 MB | high | 0 |

## Move to trash2review (medium confidence — dead code, kept for review)

| path | reason | confidence | inbound refs |
|---|---|---|---|
| `frontend/src/components/common/StateView.vue` | self-described as "replaces the ad-hoc per-view implementations" but zero components use it — created, never adopted (last commit 2026-07-26, the orphan-removal commit) | medium-high | 0 |
| `frontend/src/components/tags/TagList.vue` | tag selection moved to inline `#hashtag` typeahead in EntryEditor (`EntryEditor.vue:221-225`); component has zero importers | medium-high | 0 |
| `frontend/src/components/notes/NoteListItem.vue` | component (≠ the `NoteListItem` *type*, which lives in `types/index.ts` and is used); NotesView renders list rows itself; zero importers of the .vue | medium-high | 0 |
| `mobile/capacitor.config.ts`, `mobile/Makefile` | mobile scaffolding never realized: no `android/`/`ios/` dirs, no `@capacitor/*` in `frontend/package.json`, untouched since 2026-07-17 | medium | 0 |

## Leave alone (checked, alive)

- `types/index.ts` — imported by 10+ files via `from '../../types'` (initial
  scan false-positive; verified by hand).
- `backend/app/**/{models,routers,services,schemas}/backup.py` etc. — "backup"
  matched my filename grep but is the live auto-backup feature (scheduler armed
  in `main.py:87-114`).
- `docs/images/lifelogr_aboutus.jpg` — unreferenced, but docs images are cheap
  and blog post (`docs/blog_lifelogr_vs_diarium.md`) is recent; leaving.
  Flagged in AUDIT.md instead.
- `.claude/settings.local.json` — unreferenced but harmless local permissions
  cache; convention is to keep it (it's the user's own tooling state).
- Empty `__init__.py` files — required for package imports, not bloat.
- `__pycache__`/`.pytest_cache`/`*.pyc` — exist on disk but **untracked**
  (gitignore already covers); nothing tracked to move.

## Net effect

9 files, ~1.4 MB → `trash2review/`. No test file touched (all 423+21 still
collectable). Zero import-wiring changes needed.
