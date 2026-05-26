---
"@ai-hero/sandcastle": patch
---

Fix `sandcastle docker build-image` / `podman build-image` failing on macOS hosts. The generated Dockerfile now aligns the agent UID/GID with `groupmod -o` / `usermod -o` (`--non-unique`), so a host GID that collides with a reserved GID in `node:22-bookworm` (notably macOS's primary group `staff` = GID 20, occupied by `dialout`) no longer aborts the build with `GID '20' already exists`. Existing scaffolds need to re-run `sandcastle init` or add `-o` to the `groupmod`/`usermod` line by hand.
