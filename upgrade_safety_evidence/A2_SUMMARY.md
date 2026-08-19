# A2: docs/doc.py regeneration cross-check

## Finding + fix (unrelated to root-brownfield safety, but caught by this step)

Regenerating `docs/VARIABLES.md` from HEAD (`python3 docs/doc.py`, run from `docs/`) initially
produced a **non-empty diff** against the committed copy: several variables had switched from
referencing `deployment_path` directly to `deployment_path_final` (the trailing-slash-sanitized
version, introduced earlier in this branch) without the docs being regenerated, and 3
`kafka_connect_replicator` path variables that were converted to deployment_path-aware ternaries
also weren't reflected. This is a documentation-freshness gap, not a variable-default-safety bug
(the underlying `defaults/main.yml` values, already verified in A1, were correct all along - just
not mirrored in the generated doc). Fixed by regenerating and committing (`10445f0c0`).
After the fix, `git diff docs/VARIABLES.md` is empty - `doc.py`'s output now matches the
committed file exactly.

## Cross-check against A1

Regenerated `VARIABLES.md` from the `v7.6.12` worktree too (`A2_VARIABLES_v7.6.12.md`) and
extracted the `Default:` line for each of the 32 defaults-file-sourced path variables from A1
(the `vars/main.yml`-sourced ones like `log.dirs`/`dataDir` aren't in this doc, since `doc.py`
only parses `roles/*/defaults/main.yml`, not `vars/main.yml` - A1 already covers those directly).
Extracted each's else-branch literal and compared v7.6.12 vs. HEAD:

**32/32 matched, 0 mismatches** - independently corroborates A1's result via a completely
different extraction pathway (mechanical doc-text parsing instead of live Jinja evaluation).
