# Plan: Z AI API Provider + Gitea Issue Tracker

## Overview

Three changes to Sandcastle:

1. **Z AI GLM Coding Plan** as a new API provider selection during `sandcastle init`
2. **Gitea Issues** as a new issue tracker entry
3. **Bug fix:** `OPENAI_KEY` → `OPENAI_API_KEY` in codex `.env.example`

All changes are init-only. No modifications to `AgentProvider.ts` — env vars flow through `.env` → `EnvResolver` → sandbox as-is.

---

## 1. Z AI as API Provider

### What it is

Z AI (https://z.ai/) offers a "GLM Coding Plan" with two API endpoints:

- **Anthropic-compatible:** `https://api.z.ai/api/anthropic`
- **OpenAI-compatible:** `https://api.z.ai/api/coding/paas/v4`

Available models: `GLM-5.1`, `GLM-5`, `GLM-5-Turbo`, `GLM-4.7`, `GLM-4.5-air`. Default: `GLM-5.1`.

### Where it lives

New `ApiProviderEntry` registry in `InitService.ts`, alongside existing `AGENT_REGISTRY` and `ISSUE_TRACKER_REGISTRY`.

### Supported agents

| Agent       | Z AI Endpoint         | Env vars in `.env.example`                                                    | Default model |
| ----------- | --------------------- | ----------------------------------------------------------------------------- | ------------- |
| Claude Code | Anthropic             | `ANTHROPIC_AUTH_TOKEN=` + `ANTHROPIC_BASE_URL=https://api.z.ai/api/anthropic` | `GLM-5.1`     |
| pi          | Native `zai` provider | `ZAI_API_KEY=`                                                                | `zai/GLM-5.1` |
| Codex       | OpenAI                | `OPENAI_API_KEY=` + `OPENAI_BASE_URL=https://api.z.ai/api/coding/paas/v4`     | `GLM-5.1`     |

Z AI is **not offered** for Cursor, OpenCode, or Copilot (their auth flows don't map cleanly).

### `ApiProviderEntry` shape

```typescript
interface ApiProviderEntry {
  name: string; // "anthropic-direct", "zai"
  label: string; // "Anthropic (direct)", "Z AI"
  agentDefaults: Record<
    string,
    {
      envExample: string; // lines for .env.example
      defaultModel: string;
    }
  >;
}
```

### Init flow order (updated)

Agent → **API provider** → Model → Sandbox → Template → Issue tracker → Create label? → Build image? → Install deps?

New CLI flag: `--api-provider` (values: `anthropic-direct`, `zai`).

When `anthropic-direct` is selected, behavior is identical to today.

### `.env.example` generation

Current behavior: agent entry owns `envExample`. Issue tracker entry appends its own.

New behavior: API provider entry overrides the agent's `envExample` when Z AI is selected. The agent's `envExample` is used when `anthropic-direct` is selected (current behavior preserved).

### Default model override

When Z AI is selected, the default model changes per the mapping above. The user can still override with `--model`.

For pi: the model string uses the `provider/model` format (`zai/GLM-5.1`) so pi explicitly routes to the Z AI provider. Pi has native `ZAI_API_KEY` support and reads it automatically.

### Validation

If the user picks an agent that's not supported by Z AI (e.g. Cursor + Z AI), init should either:

- Hide Z AI from the API provider prompt, or
- Show an error: "Z AI is not available for Cursor. Choose Anthropic (direct)."

### Base URL handling

Z AI base URLs are **pre-filled** in `.env.example` (not blank placeholders). Only the API key is a blank placeholder.

---

## 2. Gitea Issue Tracker

### New entry in `ISSUE_TRACKER_REGISTRY`

Name: `gitea-issues`
Label: `Gitea Issues`

### Dockerfile snippet (`ISSUE_TRACKER_TOOLS`)

Multi-arch curl install of the `tea` CLI binary:

```dockerfile
# Install Gitea CLI (tea)
RUN ARCH=$(dpkg --print-architecture) && \
    curl -fsSL "https://gitea.com/gitea/tea/releases/latest/download/tea-${ARCH}-linux" \
    -o /usr/local/bin/tea && chmod +x /usr/local/bin/tea
```

(Verify exact download URL pattern from https://gitea.com/gitea/tea/releases during implementation.)

### Template commands

```typescript
templateArgs: {
  LIST_TASKS_COMMAND: `tea issues list --state open --limit 100 --output json --label Sandcastle`,
  VIEW_TASK_COMMAND: "tea issues view <ID>",
  CLOSE_TASK_COMMAND: `tea issues close <ID> --comment "Completed by Sandcastle"`,
  ISSUE_TRACKER_TOOLS: GITEA_CLI_TOOLS,  // the Dockerfile snippet above
}
```

**Note:** Research the exact `tea` CLI flags during implementation. The commands should match the GitHub pattern (JSON output, label filtering, close with comment). `tea` uses repo context from `git remote`, so no explicit `--repo` flag should be needed if the repo's origin points at the Gitea instance.

### `.env.example`

```
# Gitea personal access token
GITEA_TOKEN=
# Gitea server URL (e.g. https://gitea.mycompany.com)
GITEA_SERVER_URL=
```

No `GH_TOKEN` — each tracker only asks for its own tokens.

### `--create-label` support

Gitea supports labels. The `--create-label` flag should work the same as `github-issues`, but using `tea` commands to create the `Sandcastle` label.

### `tea login` auto-injection

The `tea` CLI requires an explicit `tea login add` step (unlike `gh` which works with just `GH_TOKEN` in env).

**`InitService.rewriteMainTs`** gains logic to inject a `tea login` hook into the scaffolded `main.ts` when `gitea-issues` is the selected tracker:

```typescript
{
  command: 'tea login add --name sandcastle --url "$GITEA_SERVER_URL" --token "$GITEA_TOKEN" || true';
}
```

This is appended to the existing `onSandboxReady` hooks array. The `|| true` handles re-runs (login already exists).

### GitHub `.env.example` cross-pollution

`github-issues` and `gitea-issues` do **not** share tokens. Each entry only includes its own env vars:

- `github-issues` → `GH_TOKEN=`
- `gitea-issues` → `GITEA_TOKEN=` + `GITEA_SERVER_URL=`

---

## 3. Bug Fix: Codex env var name

Current `.env.example` for codex says `OPENAI_KEY=`. The Codex CLI reads `OPENAI_API_KEY`.

Fix: change the `envExample` in the codex `AgentEntry` from `OPENAI_KEY=` to `OPENAI_API_KEY=`.

Also update any test references that assert `OPENAI_KEY`.

---

## Files to modify

| File                        | Change                                                                                                                                                                                                      |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/InitService.ts`        | Add `ApiProviderEntry` registry, `gitea-issues` issue tracker entry, `tea login` hook injection in `rewriteMainTs`, new `--api-provider` CLI flag, updated init flow order, `.env.example` generation logic |
| `src/InitService.test.ts`   | Tests for Z AI provider selection, Gitea tracker, `tea login` injection                                                                                                                                     |
| `src/AgentProvider.test.ts` | Update `OPENAI_KEY` test references to `OPENAI_API_KEY`                                                                                                                                                     |
| `README.md`                 | Document `--api-provider` flag, `gitea-issues` tracker, updated init flow                                                                                                                                   |

## Files NOT modified

| File                   | Why                                                          |
| ---------------------- | ------------------------------------------------------------ |
| `src/AgentProvider.ts` | No changes needed — env vars flow through `.env` passthrough |
| `src/EnvResolver.ts`   | No changes — already generic key-value passthrough           |
| `src/index.ts`         | No new exports                                               |
| `CONTEXT.md`           | No new domain concepts                                       |
| `docs/adr/`            | No ADRs needed — follows existing patterns                   |

---

## Open questions for implementation

1. **Exact `tea` CLI flags** — verify `tea issues list --output json` produces usable JSON, and what its schema looks like. The prompt templates may need adjustment if the shape differs from `gh issue list --json`.
2. **`tea` download URL** — confirm the exact URL pattern from https://gitea.com/gitea/tea/releases for multi-arch binary downloads.
3. **`tea login add` flag for non-interactive** — confirm `tea login add --name <name> --url <url> --token <token>` works non-interactively (no prompts).
4. **Z AI model names in Sandcastle** — Z AI docs show uppercase `GLM-5.1`, `GLM-4.7`, etc. Verify pi and Codex accept these model names as-is (Claude Code uses them through the Anthropic endpoint, so model names are Z AI's responsibility).
