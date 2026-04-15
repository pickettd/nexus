# OpenAI Custom Base URL + Model Override Support — Exploration

**Status:** exploration / not implemented
**Date:** 2026-04-15
**Goal:** support `LLM_PROVIDER=openai` with:

- `OPENAI_BASE_URL`
- `OPENAI_ROUTER_MODEL`
- `OPENAI_AGENT_MODEL`
- `OPENAI_WORK_MODEL`
- `OPENAI_EMBEDDING_MODEL`

This note describes the smallest code change that would make that work in the repo's main local-app path without expanding scope into UI or database-backed settings work.

---

## TL;DR

The smallest viable change is mostly plumbing:

1. Add new OpenAI env vars to `src/config.ts`.
2. Add an `getOpenAIModelOverrides()` helper in `src/adapters/providers/factory.ts`.
3. Change the `openai` factory case to pass `baseURL` and `modelOverrides` into the existing `OpenAIProvider` constructor.
4. Add tests and docs for the new env vars.

No change is required in `src/adapters/providers/openai.ts` itself for this env-driven v1, because it already supports both inputs.

---

## Findings

### 1. The provider class already supports both features

`src/adapters/providers/openai.ts` already has the right constructor:

```ts
constructor(
  apiKey: string,
  modelOverrides?: Partial<Record<ModelTier, string>>,
  baseURL?: string,
) {
  this.client = new OpenAI({ apiKey, ...(baseURL ? { baseURL } : {}) });
  this.modelMap = { ...DEFAULT_MODEL_MAP, ...modelOverrides };
}
```

That means:

- custom base URLs are already supported by the underlying provider wrapper
- per-tier custom model names are already supported by the provider wrapper
- model names are passed through as plain strings to the OpenAI SDK

There is no repo-side allowlist for OpenAI model names.

### 2. The missing piece is config/factory wiring

Today the `openai` case in `src/adapters/providers/factory.ts` is:

```ts
case 'openai':
  if (!apiKey) throw new Error('LLM_API_KEY is required for OpenAI provider');
  return new OpenAIProvider(apiKey);
```

So the provider's customization hooks exist, but the app never supplies values for them.

### 3. The repo already uses this exact pattern for Ollama

`src/config.ts` exposes:

- `OLLAMA_BASE_URL`
- `OLLAMA_ROUTER_MODEL`
- `OLLAMA_AGENT_MODEL`
- `OLLAMA_WORK_MODEL`
- `OLLAMA_EMBEDDING_MODEL`

`src/adapters/providers/factory.ts` then builds an override object and passes it into `new OllamaProvider(...)`.

That is the cleanest precedent to copy.

### 4. Persisted per-org model settings exist, but they are not the smallest path

The repo has DB-backed model-setting helpers in `src/settings/service.ts`:

```ts
getModelId(tier, orgId)
setModelId(tier, modelId, orgId, updatedBy)
```

But today only the Gemini provider consumes them. OpenAI does not.

Using those same DB-backed settings for OpenAI would be a larger scope change because it raises follow-up questions about:

- UI exposure
- API routes for editing model IDs
- provider parity across Gemini / OpenAI / Ollama / OpenRouter

For a smallest-change implementation, env vars are the right first step.

### 5. The current UI/setup flow does not expose these settings

The local setup flow only captures:

- provider
- API key

It does not collect base URL or per-tier model names. That is acceptable for a smallest v1 if we treat this as manual `.env` configuration.

### 6. There is one separate repo nuance worth calling out

The main local UI path uses `createLLMProvider()`, so changes in `src/adapters/providers/factory.ts` will affect the normal local app flow.

However, `src/adapters/loader.ts` still hard-wires `DefaultLLMProvider` for the default adapter profile. That is a separate inconsistency in the repo. It does not block the main local-mode implementation, but it is worth noting.

---

## Reasoning

### Why this is the smallest change

The repo already has:

- an OpenAI provider wrapper
- constructor support for `baseURL`
- constructor support for per-tier model overrides
- an established env-var naming pattern for another provider

Because of that, the smallest implementation is to wire new env vars into the existing provider constructor. That avoids touching:

- request formatting
- tool-call behavior
- embeddings behavior
- setup UI
- database-backed settings

### Why env vars are the right first step

Adding UI support would require changes across:

- `ui/index.html`
- `ui/app.js`
- `/api/setup` in `src/local/server.ts`
- `.env` writeback behavior

That is all optional for a first version. If the immediate goal is "make custom OpenAI-compatible endpoints work," env-only support gets there with fewer files and less behavioral risk.

### Why the naming should mirror Ollama

Using:

- `OPENAI_BASE_URL`
- `OPENAI_ROUTER_MODEL`
- `OPENAI_AGENT_MODEL`
- `OPENAI_WORK_MODEL`
- `OPENAI_EMBEDDING_MODEL`

keeps the mental model consistent with the existing Ollama config. That makes the change easier to understand and document.

---

## Suggested Code Changes

### 1. `src/config.ts`

Add the new env vars to the zod schema:

```ts
OPENAI_BASE_URL: z.string().optional(),
OPENAI_ROUTER_MODEL: z.string().optional(),
OPENAI_AGENT_MODEL: z.string().optional(),
OPENAI_WORK_MODEL: z.string().optional(),
OPENAI_EMBEDDING_MODEL: z.string().optional(),
```

Suggested placement: beside the existing `LLM_PROVIDER`, `GEMINI_API_KEY`, and `OLLAMA_*` fields.

### Why

- makes the new settings part of validated config
- avoids raw `process.env` reads in most of the codepath
- matches the existing configuration pattern

---

### 2. `src/adapters/providers/factory.ts`

Add an OpenAI override helper mirroring the Ollama helper:

```ts
function getOpenAIModelOverrides(): Partial<Record<ModelTier, string>> | undefined {
  const overrides: Partial<Record<ModelTier, string>> = {};

  if (config.OPENAI_ROUTER_MODEL) overrides.ROUTER = config.OPENAI_ROUTER_MODEL;
  if (config.OPENAI_AGENT_MODEL) overrides.AGENT = config.OPENAI_AGENT_MODEL;
  if (config.OPENAI_WORK_MODEL) overrides.WORK = config.OPENAI_WORK_MODEL;
  if (config.OPENAI_EMBEDDING_MODEL) overrides.EMBEDDING = config.OPENAI_EMBEDDING_MODEL;

  return Object.keys(overrides).length > 0 ? overrides : undefined;
}
```

Then change the `openai` case from:

```ts
return new OpenAIProvider(apiKey);
```

to:

```ts
return new OpenAIProvider(
  apiKey,
  getOpenAIModelOverrides(),
  config.OPENAI_BASE_URL || undefined,
);
```

### Why

- reuses existing provider behavior
- keeps all provider-construction logic in one place
- mirrors how Ollama is already handled

### Important scope decision

For the smallest change, this should affect only the `openai` provider case.

Do not expand the same config into:

- `openrouter`
- DB-backed per-org model overrides
- UI/runtime settings

Those can be follow-ups if needed.

---

### 3. `src/adapters/providers/factory.test.ts`

Add focused test coverage for the new behavior.

Suggested tests:

### Creates OpenAI provider with custom base URL

```ts
mockConfig.LLM_PROVIDER = 'openai';
mockConfig.OPENAI_BASE_URL = 'http://openai-compatible.internal/v1';
createLLMProvider();
expect(mockOpenAIProvider).toHaveBeenCalledWith(
  'test-api-key',
  undefined,
  'http://openai-compatible.internal/v1',
);
```

### Creates OpenAI provider with model overrides

```ts
mockConfig.LLM_PROVIDER = 'openai';
mockConfig.OPENAI_ROUTER_MODEL = 'qwen/qwen3-32b';
mockConfig.OPENAI_AGENT_MODEL = 'my-agent-model';
mockConfig.OPENAI_WORK_MODEL = 'my-work-model';
mockConfig.OPENAI_EMBEDDING_MODEL = 'my-embedding-model';
createLLMProvider();
expect(mockOpenAIProvider).toHaveBeenCalledWith(
  'test-api-key',
  {
    ROUTER: 'qwen/qwen3-32b',
    AGENT: 'my-agent-model',
    WORK: 'my-work-model',
    EMBEDDING: 'my-embedding-model',
  },
  undefined,
);
```

### Why

- proves the factory is wiring the new config into the provider
- keeps the change narrowly tested at the actual integration point

---

### 4. `.env.example`

Add new commented examples, similar to the existing Ollama block:

```dotenv
# OPENAI_BASE_URL=http://localhost:4000/v1
# OPENAI_ROUTER_MODEL=gpt-4.1-mini
# OPENAI_AGENT_MODEL=gpt-4.1
# OPENAI_WORK_MODEL=o3
# OPENAI_EMBEDDING_MODEL=text-embedding-3-small
```

### Why

- makes the feature discoverable
- gives users a working shape for OpenAI-compatible backends

---

### 5. `README.md`

Extend the configuration table and add one short usage example.

Suggested example:

```dotenv
LLM_PROVIDER=openai
LLM_API_KEY=your-key
OPENAI_BASE_URL=http://localhost:4000/v1
OPENAI_ROUTER_MODEL=qwen/qwen3-32b
OPENAI_AGENT_MODEL=qwen/qwen3-235b
OPENAI_WORK_MODEL=deepseek/deepseek-r1
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
```

### Why

- the feature otherwise remains invisible even after implementation
- this is the main user-facing repo documentation

---

## Optional But Recommended Follow-Up

### `src/adapters/loader.ts`

Today the default adapter loader hard-wires Gemini:

```ts
llmProvider: new DefaultLLMProvider(config.LLM_API_KEY ?? ''),
```

If repo-wide parity is important, that should be changed to use `createLLMProvider()` too.

### Why this is optional

- the main local UI path already uses `createLLMProvider()`
- the user-facing local application will benefit from the OpenAI config work without touching this file

### Why it is still recommended

- it removes an unexpected difference between two startup paths
- it makes `LLM_PROVIDER=openai` behave more consistently across the repo

---

## Non-Goals For The Smallest Slice

- no UI fields for base URL or model IDs
- no `/api/setup` changes
- no DB-backed model-override wiring for OpenAI
- no per-org runtime settings support
- no change to OpenAI default model choices
- no OpenRouter behavior changes

---

## Expected Usage After Implementation

```dotenv
LLM_PROVIDER=openai
LLM_API_KEY=sk-...
OPENAI_BASE_URL=http://localhost:4000/v1
OPENAI_ROUTER_MODEL=qwen/qwen3-32b
OPENAI_AGENT_MODEL=qwen/qwen3-235b
OPENAI_WORK_MODEL=deepseek/deepseek-r1
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
```

This should allow Nexus to talk to an OpenAI-compatible backend while still using the existing `openai` provider path.

If any of the `OPENAI_*_MODEL` vars are omitted, the current defaults in `src/adapters/providers/openai.ts` should continue to apply.

---

## Minimal File List

For the smallest useful implementation:

1. `src/config.ts`
2. `src/adapters/providers/factory.ts`
3. `src/adapters/providers/factory.test.ts`
4. `.env.example`
5. `README.md`

Optional parity follow-up:

6. `src/adapters/loader.ts`

---

## Recommendation

Implement the env-only path first.

That gets custom OpenAI-compatible endpoints and custom model names working with the smallest number of moving parts, and it does so by reusing code the repo already has instead of inventing a second configuration system.
