# COLONY-PATCHES.md — COL5148351 Custom Modifications

This document tracks all Colony-specific patches applied on top of the
upstream `NousResearch/hermes-agent` releases. Each patch is a separate
commit on the `colony` branch.

**Base version:** `v2026.4.16`
**Branch strategy:** `main` mirrors upstream (fast-forward only), `colony`
has all Colony patches (merge commits from upstream tags, never rebase).

---

## Patch 1: API Server Model Override for Two-Brain Architecture

**Commit:** `cc8291e9`
**Files changed:** `gateway/platforms/api_server.py`

### Problem

Clicky (the Colony's macOS desktop app) uses a two-brain architecture:
- **Vision model** (OpenClaw on `:14135`) — handles screen analysis via VLM
- **Agent model** (Hermes on `:8642`) — handles task execution via the API server

When Clicky selects a model in its UI (e.g. `xai/grok-4.20-multi-agent`),
the request goes through the Hermes API server's `/v1/chat/completions` endpoint.
Without a model override, the API server ignores the caller's model choice and
uses the gateway default (typically ollama). This causes 404 errors when the
selected model isn't available on the default provider.

### Solution

1. **`_resolve_provider_for_model_override()`** — New static method that resolves
   provider credentials for a model override. Checks `custom_providers` in
   `config.yaml` first (exact match on model name with provider prefix handling),
   then falls back to `detect_provider_for_model()` for built-in providers.
   Returns `(corrected_model_name, runtime_kwargs)` or `None`.

2. **`_create_agent(model_override=...)`** — New parameter. When provided, the
   override model takes precedence over the gateway default, and the correct
   provider is resolved via the method above.

3. **`_handle_models()`** — Expanded from returning a single model to enumerating
   ALL models the gateway can route to: custom_providers from config.yaml plus
   built-in provider catalogs that have credentials configured. Frontends use
   this to populate their model picker.

4. **`_handle_chat_completions()`** and **`_handle_responses()`** — Extract
   `model_override` from the request body when the caller's model differs from
   the server's advertised name, and pass it through to `_create_agent`.

5. **`_run_agent(model_override=...)`** — Passthrough parameter to `_create_agent`.

### Configuration

No new config keys required. The patch uses existing `custom_providers` entries
from `config.yaml` and built-in provider registries.

### Merge Notes

- `_create_agent` signature change: `model_override` is added as a keyword-only
  argument with default `None`, so all existing callers are backward-compatible.
- `_run_agent` signature change: same pattern, `model_override` defaulted to
  `None`.
- `_handle_models` response format: `data` is now a dynamic list instead of a
  single-element list. Backward-compatible for any client that iterates over
  `data`.

### Re-application Checklist (for future upstream merges)

- [ ] Verify `_create_agent` signature hasn't gained new required params
- [ ] Check `_handle_models` hasn't been reworked upstream with its own model
      enumeration (would supersede this patch)
- [ ] Verify `_run_agent` (formerly `_run_agent_in_thread`) passthrough
- [ ] Verify Responses API endpoint (`_handle_responses`) still exists and
      hasn't been refactored into a separate module