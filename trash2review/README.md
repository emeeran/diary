# trash2review

Holding pen for files removed during the cleanup pipeline. **Nothing here is
deleted** — each file was `git mv`'d from its original path, preserving history.

## Restore a file

```bash
git mv trash2review/<path>/<file> <original/path>/<file>
```

The mirrored directory structure matches the original repo layout, so the
restore is a mirror of the move. After restoring, re-run the test suite — the
file was moved because it appeared unused, so a restore may need its import
wiring re-added too.

## Why this exists

The pipeline never hard-deletes. Review the files here, restore any that were
moved in error, and empty this folder once you're satisfied nothing is missed.
Every move is logged in `.pipeline/purge-plan.md` with the reason and the
inbound-reference count found at decision time.
