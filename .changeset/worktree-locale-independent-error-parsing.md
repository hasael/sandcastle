---
"@ai-hero/sandcastle": patch
---

Fix worktree creation failing under non-English git locales. `WorktreeManager` matched git's human-readable stderr (e.g. "invalid reference") to decide control flow, but git localizes those strings, so in a non-English locale the new-branch fallback never fired and worktree creation broke outright. Git is now invoked with `LC_ALL=C` so its messages are always English and machine-stable.
