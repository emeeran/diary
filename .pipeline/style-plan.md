# Phase 3 Style Plan (dry run) — 2026-08-25

Convention detection (majority = the established idiom; the codebase's oldest
human-authored files set it):

## Detected conventions

| dimension | majority style | evidence |
|---|---|---|
| FE quotes/semis | single quotes, no semicolons | 88 single-quote files vs 17 double-quote+semicolon files |
| FE error extraction | local `errMsg(e)` helper | defined 10× — once per file, identical body |
| BE formatting | ruff (line-length 100), isort | configured in pyproject.toml |
| BE docstrings | one-line summary + prose paragraphs; Google-style `Args:` almost unused (1 hit) | census |
| Log/user-string tone | plain text backend; emoji only in the OAuth HTML pages (4) and emoji-picker data | census |

## Actions

### A. Backend: apply ruff format + isort repo-wide (app/)  [APPLIED]
24 files reformatted, 23 import-order fixes, 2 missing `noqa: E402` repaired.
All 423 tests pass after.

### B. Frontend: normalize the 17 double-quote/semicolon files to the
majority single-quote/no-semicolon style  [APPLY]
Files (settings panels/tabs + api/backup, api/entries, api/settings,
api/client, EntryEditor, EmojiPicker, EditorToolbar, MediaGrid,
useAttachments, SettingsView). Mechanical only — prettier-style transform,
no logic edits. Verify with `npm run build` (vue-tsc) + tests.

### C. Frontend: promote `errMsg` to one shared helper  [APPLY]
10 identical per-file definitions → one in `src/utils/errMsg.ts` (or extend
existing utils module). Files import it. Pure dedup of an existing idiom —
the helper body is already the codebase standard.

## Deliberately NOT changed

- **Emoji in OAuth success pages / emoji picker** — pre-existing human tone in
  user-facing HTML; matching it, not stripping (per "match existing tone").
- **Backend docstring style** — one-line-summary prose style is already
  consistent; no Google/NumPy conversion needed.
- **Vue SFC section order / component structure** — consistent already.
- `console.error` in openExternal — that's error logging, not debug noise;
  keep.
