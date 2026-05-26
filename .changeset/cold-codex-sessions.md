---
"@ai-hero/sandcastle": patch
---

Add resume support for the Codex agent provider and move session storage behind provider-owned session stores.

The per-provider session transfer is now owned by the provider's `sessionStorage.transfer` (ADR 0012). The free `transferSession` export is removed from the public API — agent providers apply their own format-specific `cwd` rewriting internally.
