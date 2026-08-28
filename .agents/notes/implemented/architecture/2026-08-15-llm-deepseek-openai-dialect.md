# Agent Note: llm-deepseek OpenAI dialect

Status: implemented

English | [中文](2026-08-15-llm-deepseek-openai-dialect.zh.md)

## Problem

The default `llm-deepseek` provider was bound to the DeepSeek wire dialect. Its serializer always emitted the DeepSeek extensions — the `thinking` request field, the `reasoning_content` assistant passback, and the `max` reasoning effort — which a plain OpenAI chat-completions endpoint (GPT models or an OpenAI-compatible gateway) rejects or ignores. Pointing `baseURL` at `https://api.openai.com/v1` therefore produced broken requests, and the default endpoint, credential reference, and model catalog were DeepSeek-specific with no switch.

## Decision

`Config` gains `dialect: 'deepseek' | 'openai'` (default `deepseek`). `resolveAdapterOptions` remains the one explicit resolve step and now picks per-dialect defaults: `apiKeyEnv` (`DEEPSEEK_API_KEY` / `OPENAI_API_KEY`), `baseURL` (public DeepSeek API / `https://api.openai.com/v1`, each honoring its own trusted env var), and the advisory model catalog (`deepseek-v4-flash`/`deepseek-v4-pro` / `gpt-4o`/`gpt-4o-mini`/`gpt-4.1`).

The serializer is dialect-aware: the OpenAI dialect never emits `thinking`, never passes `reasoning_content` back in assistant history, and maps effort `off`/`high` only — `max` fails loudly with `UNSUPPORTED_REASONING_EFFORT` (per request) or at resolve (in config). Configuring `thinking` with `dialect: openai` fails plugin load. The adapter's `resolveModel` exposes `off`/`high` with default `off` on the OpenAI dialect. DeepSeek streams require `[DONE]`; the OpenAI dialect also accepts EOF after a valid `finish_reason`, accommodating compatible gateways that omit the sentinel without accepting a truncated stream. During fragmented tool calls, later empty function names do not replace an earlier non-empty name. Error objects embedded in an HTTP 200 SSE response are classified as provider failures instead of being ignored until EOF.

The settings layer normalizes an absent `models` key to `[]`, which made "empty" indistinguishable from "unset"; an empty or unset `models` list now resolves to the dialect's catalog instead of advertising no models.

The Models page editor (deepseek family) offers the dialect selector in the customized fold; switching it swaps the base-URL placeholder, the inherited advisory rows, and the credential reference (`OPENAI_API_KEY` / `DEEPSEEK_API_KEY`), honoring an unsaved dialect switch in the draft so a single apply writes the dialect and stores the key under the new reference. The configurable-provider directory display name follows the dialect (`OpenAI` / `DeepSeek`) and re-registers in place on change, alongside the existing retry-policy re-registration.

## Alternatives considered

**Add a separate `llm-openai` provider plugin.** Rejected: the user asked to adapt the default provider, and the wire protocol was already OpenAI-compatible — the gap was the DeepSeek-specific surface, not transport.

**Keep the schema defaults and special-case the openai dialect.** Rejected: the settings layer's resolved value cannot distinguish a stored `models: []` from the schema default, so per-dialect defaults must live in the explicit resolve step.

**Map `max` to `high` on the OpenAI dialect.** Rejected: silent substitution makes the selected control differ from the logged request intent; fail loud instead.

## Consequences

A composition can point the `deepseek-official` route at any OpenAI-compatible chat-completions endpoint with OpenAI defaults, or keep DeepSeek unchanged. The route id stays `deepseek-official` in both dialects; only display name, catalog, defaults, and wire surface change. An empty `models` list no longer means "advertise nothing" (the settings layer cannot express that distinctly); deployments needing an empty advisory catalog must not use this adapter's catalog field. Docs updated: `packages/llm/llm-deepseek/README.md` + `.zh.md`, and the regenerated `docs/config-catalog.md` + mirrored `.zh.md`.
