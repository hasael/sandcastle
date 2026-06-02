---
"@ai-hero/sandcastle": minor
---

Add Z AI API provider and Gitea Issues tracker support

- **Z AI GLM Coding Plan**: New `--api-provider` flag during `sandcastle init` (values: `anthropic-direct`, `zai`). When `zai` is selected, env vars and default model change for supported agents (Claude Code, pi, Codex). The init flow now asks for API provider between agent and model selection.
- **Gitea Issues**: New `gitea-issues` issue tracker entry. Installs `tea` CLI in Dockerfile, auto-injects `tea login add` sandbox hook into scaffolded `main.ts`, and uses `GITEA_TOKEN` + `GITEA_SERVER_URL` env vars. Supports `--create-label` the same as GitHub Issues.
- **Bug fix**: Codex `.env.example` now correctly uses `OPENAI_API_KEY` instead of `OPENAI_KEY`.
