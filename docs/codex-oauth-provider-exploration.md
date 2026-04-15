# Codex OAuth LLM Provider — Exploration & Checklist

**Status:** exploration / not yet implemented
**Branch context:** `figure-out-codex-auth`
**Goal:** let users run Nexus with `LLM_PROVIDER=codex-oauth` so the router / agent / work tiers are powered by their ChatGPT subscription via codex-cli's OAuth credentials, with no per-token API billing.

---

## TL;DR

Two production-quality TS projects (pi-mono and opencode) already solve this, and they converge on the **same approach**:

1. Run your **own** PKCE OAuth flow against `auth.openai.com` using codex-cli's `client_id` (`app_EMoamEEZ73f0CkXaXp7hrann`).
2. Store tokens in your **own** file (not `~/.codex/auth.json`).
3. POST directly to `https://chatgpt.com/backend-api/codex/responses` over HTTP with the three magic headers.
4. Refresh with a standard `grant_type=refresh_token` grant.

**Do not** shell out to the `codex` binary. **Do not** read `~/.codex/auth.json`. Both couple you to codex-cli internals, fight file locking, and (for shell-out) add multi-second spawn latency to every router call.

This doc captures findings from both reference implementations, picks the pi-mono shape as the template for Nexus, and lays out an implementation checklist.

---

## Reference implementation: pi-mono

Repo: [`badlogic/pi-mono`](https://github.com/badlogic/pi-mono), `packages/coding-agent` + `packages/ai`.

### Shape
Codex is **its own provider** (`openai-codex-responses`), registered alongside anthropic / openai / gemini / bedrock / mistral in a provider registry. Each provider exposes the same `StreamFunction<Api, Options>` contract.

### OAuth (`packages/ai/src/utils/oauth/openai-codex.ts`)

```ts
const CLIENT_ID    = "app_EMoamEEZ73f0CkXaXp7hrann";
const AUTHORIZE_URL = "https://auth.openai.com/oauth/authorize";
const TOKEN_URL     = "https://auth.openai.com/oauth/token";
const REDIRECT_URI  = "http://localhost:1455/auth/callback";
```

- Spins up a local HTTP server on `:1455`, generates PKCE verifier/challenge, opens the user's browser, exchanges the auth code for tokens.
- Decodes the ID token JWT and extracts `payload["https://api.openai.com/auth"].chatgpt_account_id` — required on every API request.
- Authorize URL carries `codex_cli_simplified_flow=true` and `originator=pi`.
- `refreshOpenAICodexToken()` does a standard `grant_type=refresh_token` POST.

### Token storage (`packages/coding-agent/src/core/auth-storage.ts`)

- Lives at `~/.pi/agent/auth.json`.
- Guarded by `proper-lockfile` so concurrent refreshes don't race.
- `getApiKey(providerId)` auto-refreshes if the access token is expired and returns a live bearer token.

### Request path (`packages/ai/src/providers/openai-codex-responses.ts`)

POSTs to `https://chatgpt.com/backend-api/codex/responses` with:

```
Authorization: Bearer <access_token>
chatgpt-account-id: <from JWT>
originator: pi
OpenAI-Beta: responses=experimental
session_id: <session>
x-client-request-id: <session>
User-Agent: pi (<platform> <release>; <arch>)
```

Body is a standard `/v1/responses` payload: `instructions`, `input`, `tools`, `reasoning.effort`, `text.verbosity`, `include: ["reasoning.encrypted_content"]`, `prompt_cache_key`. Parses SSE manually (reads `data:` frames) and feeds events into a `processResponsesStream` helper that's **shared with the regular OpenAI provider** — so tool calls, reasoning deltas, and usage parsing are reused.

One codex-specific quirk: `response.done` / `response.completed` / `response.incomplete` events are normalized to `response.completed` before downstream processing (`mapCodexEvents`).

### Model selection
`body.model = model.id` — e.g. `gpt-5-codex`, `gpt-5.1-codex`. A `clampReasoningEffort()` helper caps effort per model (e.g. `gpt-5.1-codex-mini` → `medium`).

---

## Reference implementation: opencode

Repo: [`anomalyco/opencode`](https://github.com/anomalyco/opencode), branch `dev` (143k stars).

### Shape (different from pi-mono)
Codex is **not** a separate provider. It's an **OAuth auth method attached to the existing openai provider**. Entire integration is ~500 lines in `packages/opencode/src/plugin/codex.ts`.

### How it plugs in
1. The plugin declares `auth.provider: "openai"`.
2. The core provider loader runs `createOpenAI({ apiKey, fetch })` (Vercel AI SDK).
3. The plugin's `loader(getAuth, provider)` hook returns a custom `fetch` that:
   - Strips the dummy `Authorization` header.
   - Injects `Bearer <oauth-access>` + `ChatGPT-Account-Id`.
   - Refreshes if `currentAuth.expires < Date.now()`.
   - Rewrites the URL from `api.openai.com/v1/responses` to `chatgpt.com/backend-api/codex/responses`.
4. Filters `provider.models` to an allowlist: `gpt-5.1-codex`, `gpt-5.1-codex-max`, `gpt-5.1-codex-mini`, `gpt-5.2`, `gpt-5.2-codex`, `gpt-5.3-codex`, `gpt-5.4`, `gpt-5.4-mini`.
5. Zeros all model costs (included with subscription).

### Why it's so small
The Vercel AI SDK already has a working `/v1/responses` SSE parser, tool-call handling, streaming, retries. opencode just **hijacks the fetch layer** — everything above it (parsing, tool calls) is unchanged.

### Model allowlist (useful reference for us)
```
gpt-5.1-codex, gpt-5.1-codex-max, gpt-5.1-codex-mini,
gpt-5.2, gpt-5.2-codex,
gpt-5.3-codex,
gpt-5.4, gpt-5.4-mini
```

### Auth + device flow
Two login paths: browser PKCE on localhost:1455 (same as pi-mono), **plus** a headless device-code flow via `/api/accounts/deviceauth/usercode` for SSH / container environments.

---

## Comparison

| Dimension | Shell out (`codex exec`) | opencode (fetch-hijack plugin) | pi-mono (dedicated provider) |
|---|---|---|---|
| Latency | ~2-5s subprocess spawn per call | HTTP only | HTTP only |
| Streaming | Not really | Yes (via SDK) | Yes (manual SSE) |
| Tool calls | Broken / text-only | Yes (via SDK) | Yes (shared parser) |
| LOC to add | ~80 | ~500 (one file) | ~800 split across provider + oauth + token store |
| Requires underlying SDK? | No | Yes (Vercel AI SDK) | No |
| Fits Nexus architecture? | Yes but unusably slow | **No** — Nexus doesn't use Vercel AI SDK | **Yes** |
| Embeddings | None | None | None |
| Couples to codex-cli internals? | Yes (binary path, flags, output format) | No | No |

**Conclusion:** pi-mono shape fits Nexus. The `LLMProvider` interface at `src/adapters/interfaces/llm-provider.ts:43` and the factory pattern at `src/adapters/providers/factory.ts:21` already expect "one class per provider", and Nexus isn't built on top of an SDK whose fetch layer we could hijack.

---

## Target architecture for Nexus

### Wire-up
1. New provider class `CodexOAuthProvider implements LLMProvider` in `src/adapters/providers/codex-oauth/provider.ts`.
2. OAuth + token store in the same subdirectory: `codex-oauth/auth.ts`, `codex-oauth/token-store.ts`.
3. Factory case added at `src/adapters/providers/factory.ts:21`:
   ```ts
   case 'codex-oauth':
     return new CodexOAuthProvider();
   ```
4. CLI wizard entry added at `bin/cli.ts:16`:
   ```ts
   { value: 'codex-oauth', label: 'ChatGPT (Codex OAuth, no API key)', keyUrl: '' },
   ```
   and the `provider.value !== 'ollama'` guard at `bin/cli.ts:75` extended to also skip the API key prompt for `codex-oauth`, instead triggering the PKCE browser flow.
5. Token file stored at `~/.nexus/codex-auth.json` (or wherever the project conventionally keeps local state — verify by checking the embedded PGlite / `./data/` layout mentioned in `CLAUDE.md`).

### How it maps onto `LLMProvider`
- `generateText`: build an `/v1/responses` payload from `systemInstruction` + `contents`, POST with the three headers, accumulate SSE `response.output_text.delta` events, return the final string. No streaming surface needed — the interface returns `Promise<string>`.
- `generateWithTools`: same thing, but include `tools` in the body. Parse `response.function_call` / `response.function_call_arguments.delta` events into `LLMToolCallResult.functionCalls`.
- `embedText`: return `null`. Codex endpoint has no embeddings. Users who need embeddings must run `LLM_PROVIDER=multi` with `LLM_EMBEDDING_PROVIDER=ollama` (or openai / gemini with an API key).

### Headers we will send
```
Authorization:       Bearer <access_token>
chatgpt-account-id:  <from JWT>
originator:          nexus          ← NOT codex_cli
OpenAI-Beta:         responses=experimental
session_id:          <per-process uuid>
User-Agent:          nexus/<version> (<platform> <release>; <arch>)
```

Using `originator: nexus` (not `codex_cli`) matches both pi-mono (`originator: pi`) and opencode (`originator: opencode`). OpenAI has historically policed `codex_cli` coming from non-codex clients — stay clean.

---

## Implementation checklist

Work top-to-bottom. Each box should be doable in one sitting.

### Phase 0 — verify assumptions (no code yet)
- [ ] Read the current `OpenAIProvider` at `src/adapters/providers/openai.ts` end-to-end and confirm it uses the `openai` npm SDK's `chat.completions.create()` — NOT `responses.create()`. Decide whether to use the SDK's `responses.create()` (cleaner) or hand-rolled `fetch` to `chatgpt.com/backend-api/codex/responses` (more control, matches pi-mono).
- [ ] Grep every `generateWithTools` call site and note which ones *actually rely on* `functionCalls` being populated. This tells us the blast radius if tool-call parsing has bugs in v0.
- [ ] Check where Nexus writes other local state (PGlite at `./data/pglite/`, `.env` at repo root) — decide final path for `codex-auth.json`. Candidates: `./data/codex-auth.json`, `~/.nexus/codex-auth.json`, XDG `$XDG_CONFIG_HOME/nexus/codex-auth.json`.
- [ ] Decide: do we vendor `proper-lockfile` for the token store, or is there already a lock primitive in the repo? (`proper-lockfile` is ~25KB, zero deps — likely fine to add.)
- [ ] Pin the model list. Start with the opencode allowlist above, but verify against the current `codex` CLI which models it actually ships (run `codex --help` or check its default config).

### Phase 1 — OAuth flow (standalone, testable in isolation)
- [ ] Create `src/adapters/providers/codex-oauth/auth.ts` with:
  - [ ] `generatePKCE()` — verifier (random 43-128 char base64url) + challenge (`sha256(verifier)` base64url).
  - [ ] `startOAuthFlow()` — spins up `http.createServer` on `:1455`, opens browser via `open` npm package (or falls back to printing the URL), waits for `/auth/callback?code=...&state=...`, exchanges code+verifier for tokens at `https://auth.openai.com/oauth/token`.
  - [ ] `refreshToken(refreshToken)` — standard `grant_type=refresh_token` POST.
  - [ ] `extractAccountId(idToken)` — base64-decode the middle segment of the JWT, parse JSON, read `payload["https://api.openai.com/auth"].chatgpt_account_id`. Fall back to `chatgpt_account_id` top-level and `organizations[0].id` like opencode does.
- [ ] Hard-code the constants from pi-mono/opencode:
  ```ts
  const CLIENT_ID    = "app_EMoamEEZ73f0CkXaXp7hrann";
  const AUTHORIZE_URL = "https://auth.openai.com/oauth/authorize";
  const TOKEN_URL     = "https://auth.openai.com/oauth/token";
  const REDIRECT_URI  = "http://localhost:1455/auth/callback";
  const ORIGINATOR    = "nexus";
  ```
- [ ] Write a manual smoke-test script (`scripts/codex-oauth-login.ts` or similar) that runs the flow end-to-end and prints the access token + account ID. Gate it behind a dev-only entry so it doesn't ship.

### Phase 2 — token store
- [ ] Create `src/adapters/providers/codex-oauth/token-store.ts` with:
  - [ ] `load()` / `save(auth)` — JSON file at the path chosen in Phase 0.
  - [ ] `getActiveAccessToken()` — loads tokens, checks `expires_at < Date.now() + 30s` buffer, auto-refreshes if stale, saves refreshed tokens back, returns the live bearer.
  - [ ] File lock around read-modify-write via `proper-lockfile`.
  - [ ] File mode `0600` on write.
- [ ] Unit test: concurrent `getActiveAccessToken()` calls only refresh once. (Mock the refresh HTTP call; assert it's invoked exactly 1×.)

### Phase 3 — provider class
- [ ] Create `src/adapters/providers/codex-oauth/provider.ts` exporting `CodexOAuthProvider implements LLMProvider`.
- [ ] `generateText()`:
  - [ ] Call `tokenStore.getActiveAccessToken()`.
  - [ ] Build payload: `model`, `instructions: systemInstruction`, `input: [...contents as responses messages]`.
  - [ ] POST to `https://chatgpt.com/backend-api/codex/responses` with the six headers above.
  - [ ] Parse the SSE response: read `data:` frames, JSON-parse each, accumulate `response.output_text.delta` events, stop on `response.completed` / `response.done` / `response.incomplete`.
  - [ ] Return accumulated text.
  - [ ] Wrap in `withRetry()` like the other providers (see `src/adapters/providers/retry.ts`).
- [ ] `generateWithTools()`:
  - [ ] Same as above, but include `tools` in body mapped from `LLMFunctionDeclaration[]`.
  - [ ] Parse `response.output_item.added` events with `type: "function_call"` and accumulate `arguments` from `response.function_call_arguments.delta` events.
  - [ ] Return `{ text, functionCalls, raw }` shape matching `LLMToolCallResult`.
- [ ] `embedText()`: `return null;`
- [ ] Add model tier defaults:
  ```ts
  const DEFAULT_MODEL_MAP: Record<ModelTier, string> = {
    ROUTER:    'gpt-5.1-codex-mini',
    AGENT:     'gpt-5.1-codex',
    WORK:      'gpt-5.1-codex-max',
    EMBEDDING: '',  // unsupported
  };
  ```
  (Adjust once Phase 0 model-pinning is done.)
- [ ] Support env-var model overrides mirroring the Ollama pattern at `src/adapters/providers/factory.ts:10-19`:
  `CODEX_ROUTER_MODEL`, `CODEX_AGENT_MODEL`, `CODEX_WORK_MODEL`.

### Phase 4 — wire-up
- [ ] Add `CODEX_*_MODEL` fields to the zod schema at `src/config.ts:8`.
- [ ] Add factory case at `src/adapters/providers/factory.ts:21` — construct `CodexOAuthProvider` with model overrides from config.
- [ ] If the token file doesn't exist on first use, the provider should throw a clear error telling the user to run the login flow (don't silently fail mid-request).
- [ ] Add CLI wizard entry at `bin/cli.ts:16` and extend the ollama-branch guard at `bin/cli.ts:75` to also handle `codex-oauth` — on selection, run the PKCE flow directly from the wizard instead of prompting for an API key.
- [ ] Update `.env.example` with the new provider option and a comment pointing at the login flow.
- [ ] Update `CLAUDE.md` — add `codex-oauth` to the `LLM_PROVIDER` table row at `CLAUDE.md:27`.
- [ ] Update `README.md` with a "Using your ChatGPT subscription" section.

### Phase 5 — testing & validation
- [ ] Unit tests for SSE parsing (mock `fetch`, feed it canned event streams).
- [ ] Unit tests for JWT account-id extraction with all three fallback paths.
- [ ] Unit tests for token-refresh triggering on expiry.
- [ ] Integration test (skipped in CI, gated on env var): run `generateText` against a real token, assert non-empty response.
- [ ] Manual test: full chat session in the local UI with `LLM_PROVIDER=codex-oauth`.
- [ ] Manual test: agent with tool use (router → agent → work). Confirm tool calls parse correctly.
- [ ] Manual test: `LLM_PROVIDER=multi` with `LLM_AGENT_PROVIDER=codex-oauth` and `LLM_EMBEDDING_PROVIDER=ollama`. This is probably the realistic deployment.
- [ ] Measure latency vs. direct `anthropic` / `openai` — note in the PR description.

---

## Open questions / risks

1. **Endpoint stability.** `chatgpt.com/backend-api/codex/responses` is a private endpoint. OpenAI could change payload shape, rename events, or tighten `originator`/`User-Agent` validation at any time. We pin nothing and have no SLA.
2. **Rate limits.** ChatGPT subscription rate limits are tier-based and opaque. Nexus's router-tier calls are chatty (one per message). Pro subscribers should be fine, Plus may hit limits fast.
3. **Tool-call event shape.** The `/v1/responses` API has multiple event types for function calling that evolved over time. Need to verify against a real capture before trusting any schema.
4. **Embeddings gap.** Parts of Nexus probably depend on `embedText()` returning real vectors. Grep for `.embedText(` call sites and confirm `null` is a graceful fallback, or document the `multi` provider workaround loudly.
5. **Port 1455 already in use.** The OAuth callback port is hardcoded by the OAuth provider config — we can't change it. If another codex-cli login is in progress, ours will fail. Detect and print a clear error.
6. **Token revocation.** If the user revokes the token in ChatGPT settings, refresh will 401. Catch this and prompt re-login rather than retrying forever.
7. **Concurrent Nexus instances.** Two running servers sharing one token file. The file lock handles refresh races, but if both are under heavy load, they'll fight. Probably fine; worth noting.
8. **Legal / ToS.** Using ChatGPT-subscription credentials from a non-codex-cli binary may violate OpenAI's ToS depending on how narrowly "codex cli" is interpreted. pi-mono and opencode both do it openly, but that's not legal cover. Worth a 5-minute read of the relevant ToS section before shipping.

---

## References

- pi-mono: https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent
  - OAuth: `packages/ai/src/utils/oauth/openai-codex.ts`
  - Provider: `packages/ai/src/providers/openai-codex-responses.ts`
  - Shared parser: `packages/ai/src/providers/openai-responses-shared.ts`
  - Token store: `packages/coding-agent/src/core/auth-storage.ts`
- opencode: https://github.com/anomalyco/opencode/tree/dev
  - Full integration: `packages/opencode/src/plugin/codex.ts`
  - Generic auth store: `packages/opencode/src/auth/index.ts`
  - Provider loader: `packages/opencode/src/provider/provider.ts`
- Nexus provider contract: `src/adapters/interfaces/llm-provider.ts:43`
- Nexus provider factory: `src/adapters/providers/factory.ts:21`
- Nexus OpenAI provider (closest analog): `src/adapters/providers/openai.ts`
