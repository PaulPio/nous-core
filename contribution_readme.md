# OpenRouter Model Provider

## Solution Approach

### Analysis
Nous lacked a selectable OpenRouter provider. OpenRouter is an OpenAI Chat Completions-compatible aggregator (500+ models, Bearer auth), so it is implemented by reusing the existing `ChatCompletionsProvider` — no changes to core interfaces.

### Proposed Solution
Add a certified provider **leaf** under `self/subcortex/providers/src/providers/openrouter/`, mirroring the merged **Groq** leaf (the canonical OpenAI-compatible cloud reference). The maintainer's registry-driven refactor on the integration branch means the leaf alone surfaces OpenRouter in API-key settings and auto-discovers its full model catalog — no app/server/UI code.

### Implementation Plan (UMPIRE)

**Understand:** Nous had no OpenRouter provider. OpenAI-compatible → reuse `ChatCompletionsProvider` (`src/protocols/openai-api/`) without touching `IModelProvider` or `TextModelInputSchema`.

**Match:** Reuse `ChatCompletionsProvider`; mirror the leaf structure of `providers/groq/` (OpenAI-compatible cloud reference) and `providers/openai/`; register via the `generate:providers` codegen script; rely on the registry-driven `provider-model-discovery.ts` for model listing.

**Plan:**
1. Create `src/providers/openrouter/` — `definition.ts` (`OPENROUTER_PROVIDER_DEFINITION`, `vendorKey: 'openrouter'`, **no hand-authored UUID** — derived from `vendorKey`), `adapter.ts` (re-export `chatCompletionsAdapter`), `provider.ts` (factory → `ChatCompletionsProvider`), `index.ts` (barrel).
2. Run `pnpm --filter @nous/subcortex-providers run generate:providers` to regenerate catalogs.
3. Add `src/__tests__/providers/openrouter.test.ts`; update the existing aggregate tests for the new vendor.
4. Verify: `check:generated`, tests, typecheck, lint, build.

**Implement:** Branch `feat/openrouter-provider-leaf`, based on the integration branch `feat/contributor-friendly-inference-provider-surface` (PR target per maintainer). Directory: `self/subcortex/providers/src/providers/openrouter/`.

**Review:**
- [x] `definition.ts` does **not** hand-author `wellKnownProviderId` — built-in IDs derive from `vendorKey` via `provider-identity.ts` (corrects the original draft).
- [x] `generate:providers` added the vendor to `provider-factories.ts`, `provider-definitions.ts`, and `provider-adapters.ts`.
- [x] Core interfaces untouched (`IModelProvider`, `TextModelInputSchema`); no `@nous/shared` edits.

**Evaluate:** `check:generated` (in sync), provider tests, shared-server discovery/preferences tests, typecheck, lint, build — all green (see Testing Strategy).

---

## Testing Strategy

### Unit Tests (`src/__tests__/providers/openrouter.test.ts`)
- [x] Definition metadata correct: `vendorKey`/`displayName`/`protocol:'chat-completions'`/`adapterKey:'chat-completions'`/`defaultEndpoint:'https://openrouter.ai/api'`/`defaultModelId:'openrouter/auto'`/`auth.envVar:'OPENROUTER_API_KEY'`/`auth.header:{name:'Authorization',scheme:'bearer'}`/`modelListEndpoint:'/v1/models'`/`modelListFormat:'openai-models'`.
- [x] `healthCheckEndpoint:'/v1/key'` for authenticated key validation (post-review).
- [x] Capabilities advertise `streaming` + `modelListing`; `nativeToolUse` intentionally absent (pending #390 tool-use bridge).
- [x] No hand-authored `wellKnownProviderId`; hydrated definition passes `ProviderDefinitionSchema`.
- [x] Factory builds a `ChatCompletionsProvider` for vendor `openrouter` (config id via `deriveBuiltInProviderId('openrouter')`).
- [x] Factory throws when no OpenRouter key is available (post-review).
- [x] Factory does **not** fall back to `OPENAI_API_KEY` when `OPENROUTER_API_KEY` is missing (post-review).

### Integration Tests (existing aggregate suites, updated for the new vendor)
- [x] `provider-codegen.test.ts` — leaf discovered; generated catalogs in sync.
- [x] `provider-definitions.test.ts` — `PROVIDER_DEFINITIONS` includes `openrouter` with correct endpoint/model/envVar.
- [x] `provider-definition-types.test.ts` — `ProviderVendorKey`/`BootstrapProviderKey` unions include `openrouter`.
- [x] `provider-pipeline-integration.test.ts` — registry constructs OpenRouter as `ChatCompletionsProvider` from definition + `OPENROUTER_API_KEY`.
- [x] `adapter-resolver.test.ts` — `ADAPTER_MODULES` aggregation (also fixed a pre-existing staleness; see Notes).
- [x] `provider-registry.test.ts` — OpenRouter endpoint routing uses `OPENROUTER_API_KEY`, not `OPENAI_API_KEY` (post-review).
- [x] Shared-server `provider-model-discovery` — OpenRouter-shaped model list parsing + `testProviderApiKey` against `/v1/key` (post-review).
- [x] Shared-server `preferences-router` tests stay green — generic/definition-driven; OpenRouter needs no per-vendor UI code.

Results (final iteration): provider package **331 passed / 2 skipped**; shared-server discovery **8 passed** (incl. key-validation cases).

### Manual Testing (`pnpm dev:web`, port 4317)
- [x] OpenRouter appears in Settings → API Keys provider dropdown (leaf registration confirmed).
- [x] API key stored and the integration connects.
- [x] **Model picker lists OpenRouter's full catalog** after a small fix to an over-strict shared discovery parser (see "Model discovery fix" below). Before the fix it fell back to only `openrouter/auto`, which auto-routes to a GPT model.
- [x] Invalid OpenRouter keys fail Settings validation after `/v1/key` health-check endpoint was added (post-review).

---

## Implementation Notes

### Progress
1. **Initial PR:** Implemented the OpenRouter leaf mirroring the merged Groq leaf; regenerated catalogs; added/updated tests; model-discovery compatibility fix; full local verification green.
2. **Review iteration:** Addressed maintainer's three PR-local change requests (fail-closed factory, `/v1/key` validation, untrack scratch notes).
3. **Outcome:** PR #410 **merged** into `feat/contributor-friendly-inference-provider-surface` on 2026-06-30 by maintainer `@atlamors`.

### Key decisions & challenges
- **Targeted the integration branch.** Per maintainer guidance the PR targets `feat/contributor-friendly-inference-provider-surface`, which carries the new provider surface (`ProviderDefinitionLeaf`, IDs derived from `vendorKey`, registry-driven model discovery). Worked on a fresh `feat/openrouter-provider-leaf` cut from that tip.
- **No hand-authored UUID / no manual `index.ts` export** (corrects the original draft): IDs derive from `vendorKey`; the definition is surfaced transitively via the regenerated `provider-definitions.ts` (same as Groq).
- **Mostly leaf-driven:** the maintainer's discovery refactor (`provider-model-discovery.ts` + generic `preferences.ts` + dynamic `ApiKeysPage.tsx`) surfaces OpenRouter's API-key entry automatically from the definition's `auth.header` + `modelListEndpoint` + `modelListFormat`.
- **Model discovery fix (the one non-leaf change):** the shared `openai-models` parser required per-item `object`/`owned_by` and a top-level `object` that OpenRouter's `/v1/models` omits, so discovery fell back to only `openrouter/auto`. Made those three fields `.optional()` in `provider-model-discovery.ts` (OpenAI still validates; OpenRouter now lists its full catalog) and added an OpenRouter-shaped discovery test. Confirmed first-hand via `pnpm dev:web`. Maintainer appreciated this as a real compatibility improvement.
- **Fail-closed factory (review round 2):** `ChatCompletionsProvider` still falls back to `OPENAI_API_KEY` internally when `options.apiKey` is undefined. The OpenRouter factory now resolves only `OPENROUTER_API_KEY` (or an explicit option) and throws before delegating — a PR-local workaround until shared protocol cleanup (#413).
- **Key validation vs model listing (review round 2):** OpenRouter's `/v1/models` is public (200 even without auth). Added `healthCheckEndpoint: '/v1/key'` so Settings key testing hits an authenticated endpoint (401 for invalid keys). Model catalog discovery still uses `/v1/models`.
- **Pre-existing test staleness found & fixed:** `adapter-resolver.test.ts > aggregates all canonical adapter modules` was already failing on the integration tip (the llama-cpp leaf added a `chat-completions` provider without updating the expected list). Verified by stashing my changes and running it on the pristine tip. Updated it to reflect all four `chat-completions` leaves (groq, llama-cpp, openai, openrouter).
- **Maintainer-side merge churn:** Generated catalog and global roster-test conflicts from parallel provider leaves landing on the same integration branch were resolved by the maintainer during merge (#414), not pushed back to the contributor.
- **Pre-existing typecheck break (flagged, NOT touched):** `@nous/shared-server` typecheck fails on the integration tip at `bootstrap.ts:1335` (`cliSessionManager` not in `PrincipalSystemGatewayRuntimeDeps`) — unrelated to this work, confirmed present with my changes stashed. Reported, not fixed.
- **Windows line endings:** regenerated catalogs briefly became CRLF locally after a git stash round-trip (`core.autocrlf=true`); re-running the generator restored LF. Committed bytes are LF, matching the repo, so CI is unaffected.
- **Local assignment notes:** `contribution_readme.md` was untracked from git (`git rm --cached`) and re-ignored via `/*.md` so it stays local for this assignment without shipping process notes in the repo.

### Code Changes

**Initial PR:**
- **Files added:** `providers/openrouter/{definition,adapter,provider,index}.ts`; `__tests__/providers/openrouter.test.ts`.
- **Files modified (regenerated):** `provider-adapters.ts`, `provider-definitions.ts`, `provider-factories.ts`.
- **Files modified (discovery fix):** `self/apps/shared-server/src/provider-model-discovery.ts` (3 `.optional()` on the `openai-models` schema) + `self/apps/shared-server/__tests__/provider-model-discovery.test.ts` (new OpenRouter-shape case).
- **Tests updated:** `adapter-resolver.test.ts`, `provider-codegen.test.ts`, `provider-definitions/provider-definition-types.test.ts`, `provider-definitions/provider-definitions.test.ts`, `provider-pipeline-integration.test.ts`.

**Review iteration (merged):**
- **Files modified:** `providers/openrouter/provider.ts` (fail-closed factory), `providers/openrouter/definition.ts` (`healthCheckEndpoint: '/v1/key'`).
- **Tests added/updated:** `openrouter.test.ts`, `provider-model-discovery.test.ts`, `provider-registry.test.ts`.
- **Repo hygiene:** untracked `contribution_readme.md`; removed `!/contribution_readme.md` from `.gitignore`.

---

## Pull Request

**PR Link:** https://github.com/orthogonalhq/nous-core/pull/410  
**Target branch:** `feat/contributor-friendly-inference-provider-surface`  
**Issue closed:** #306  
**Status:** **MERGED** (2026-06-30) by `@atlamors`  
**Merge commit:** `2c52e6b`

### PR Description (final)
> Adds a certified OpenRouter provider leaf (`self/subcortex/providers/src/providers/openrouter/`). OpenRouter is OpenAI Chat Completions-compatible, so the leaf carries only OpenRouter metadata and reuses the shared `ChatCompletionsProvider`, mirroring the Groq leaf. Built-in ID derives from `vendorKey`; catalogs regenerated via `generate:providers`. OpenRouter appears in API-key settings and its full model catalog is discovered via the registry-driven model discovery.
>
> **Model discovery fix:** the shared `openai-models` parser required fields OpenRouter's `/v1/models` omits; made them optional so the full catalog lists. **Review fixes:** fail-closed factory (no `OPENAI_API_KEY` fallback), key validation via `/v1/key`, removed scratch notes from repo.

### Maintainer Feedback

**Round 1 — initial review** (`@atlamors`, early-access provider integration review)

Positive:
- Provider-leaf shape looks strong: certified structure, shared Chat Completions path, `OPENROUTER_API_KEY`, Bearer auth metadata, Settings/API Keys surfacing, useful provider tests.
- OpenRouter-shaped model discovery parser fix appreciated as a real compatibility improvement.

Requested changes (3 PR-local items):
1. **Prevent OpenRouter from falling back to `OPENAI_API_KEY`.** Factory passed `apiKey: options?.apiKey` into `ChatCompletionsProvider`, which can fall back to `process.env.OPENAI_API_KEY` — could send OpenAI credentials to `https://openrouter.ai/api`. Fix: fail closed at the OpenRouter factory boundary until shared protocol cleanup.
2. **Revisit API-key validation.** `/v1/models` returns 200 even without valid auth, so Settings can falsely mark invalid keys as valid. Fix: point validation at an endpoint that proves the key (or make behavior explicit).
3. **Remove scratch contribution notes.** `contribution_readme.md` and the `.gitignore` exception are local process notes, not repo content.

Non-blocking maintainer follow-ups noted: `nativeToolUse` (#390), broader protocol/adapter cleanup (#413).

**Contributor response:** Acknowledged feedback; implemented all three items in a follow-up commit.

**Round 2 — approval & merge** (`@atlamors`, 2026-06-30)

> Thanks again for the update and quick iteration. I merged this as the initial early-access OpenRouter provider leaf. The requested PR-local changes were addressed: OpenRouter now fails closed instead of falling back to `OPENAI_API_KEY`, key validation uses `/v1/key` while model discovery remains on `/v1/models`, and the scratch contribution notes / `.gitignore` exception were removed.
>
> The remaining generated catalog and global roster-test conflicts were maintainer-side merge churn from several provider leaves landing in parallel. I resolved those during merge and verified the focused provider/shared-server checks.
>
> Thanks again for the OpenRouter implementation and the model discovery compatibility work.

---

## Model discovery fix (resolved in this PR)

**Symptom (before fix):** With a valid OpenRouter key, the model picker listed only `openrouter/auto` (which auto-routes to a GPT model). OpenRouter's full catalog never appeared, so a specific model couldn't be selected.

**Root cause:** The shared discovery parser `self/apps/shared-server/src/provider-model-discovery.ts` validates `modelListFormat: 'openai-models'` responses with a strict schema that **requires** a top-level `object` and per-item `object` + `owned_by`. OpenRouter's `GET https://openrouter.ai/api/v1/models` omits those fields.

**Fix applied:** made `object`/`owned_by` (per item) and the top-level `object` `.optional()` in `OpenAIModelsResponseSchema`. Covered by an OpenRouter-shape case in `provider-model-discovery.test.ts`.

---

## Key validation fix (review round 2)

**Symptom:** Settings "Test API Key" could report success for invalid OpenRouter keys because `/v1/models` is public.

**Fix:** Added `healthCheckEndpoint: '/v1/key'` to the OpenRouter definition. `testProviderApiKey` already prefers `healthCheckEndpoint` over `modelListEndpoint`, so validation now hits `https://openrouter.ai/api/v1/key` (401 invalid, 200 valid) while catalog discovery stays on `/v1/models`.

---

## Learnings & Reflections

### The open-source contribution loop (what this PR demonstrates)

This contribution followed the full loop maintainers expect from early-access integrators:

1. **Research & align** — Read the integration branch architecture (registry-driven provider leaves, Groq as reference) before coding; target the maintainer's branch, not `main`.
2. **Submit a focused PR** — Leaf-only provider where possible; one small shared-server compatibility fix with tests and a clear rationale.
3. **Receive substantive review** — Maintainer approved the shape but caught real issues: credential fallback risk, false-positive key validation, and repo hygiene.
4. **Iterate quickly** — Address each requested change with tests, not debate; reply acknowledging feedback.
5. **Merge & handoff** — Maintainer merged, resolved parallel-leaf merge churn themselves, and tracked broader cleanup in separate issues.

### Teachable insights for future cohorts

**1. "OpenAI-compatible" ≠ "OpenAI-identical."**  
OpenRouter speaks the Chat Completions protocol but its `/v1/models` payload omits fields OpenAI always sends. Strict Zod schemas that worked for OpenAI silently broke discovery for OpenRouter. When integrating aggregators, validate against *their* responses, not the reference vendor's.

**2. Separate "list models" from "prove credentials."**  
A public model catalog endpoint is convenient for discovery but useless for key validation. Use `healthCheckEndpoint` (or equivalent) for auth proof and `modelListEndpoint` for catalog — same provider, different jobs.

**3. Shared abstractions leak vendor assumptions.**  
Reusing `ChatCompletionsProvider` was correct, but its internal `OPENAI_API_KEY` fallback is OpenAI-specific. When adding a new vendor through a shared factory, **fail closed at the leaf** if the shared layer has legacy fallbacks the maintainer hasn't cleaned up yet (#413).

**4. Flag pre-existing breaks; don't scope-creep.**  
The integration branch had an unrelated `bootstrap.ts` typecheck failure. Reporting it built trust; fixing it would have expanded the PR and mixed concerns.

**5. Process docs belong outside the repo.**  
Maintainers merge code, not assignment journals. Keep contribution write-ups local (gitignored) unless the project asks for them in `CONTRIBUTING.md` format.

**6. Parallel contributions create merge churn — that's normal.**  
Multiple provider leaves landing on one integration branch caused generated-file conflicts. The maintainer resolved those at merge time (#414). Contributors should fix conflicts *they* introduce; integration-branch reconciliation is often maintainer-owned.

---

## Second contribution cycle (what's next)

With PR #410 merged, the OpenRouter leaf is on `feat/contributor-friendly-inference-provider-surface` awaiting that branch's merge to `main`. Possible follow-ups (not started):

| Area | Issue | Notes |
|------|-------|-------|
| Native tool use bridge | #390 | OpenRouter intentionally omits `nativeToolUse` until shared bridge supports full tool loop |
| Protocol/adapter capability cleanup | #413 | Shared `ChatCompletionsProvider` OpenAI fallbacks; broader capability-source alignment |
| Integration-branch merge resolution | #414 | Maintainer tracked parallel provider-leaf churn |
| Rebase/sync fork | — | Pull upstream integration branch after merge; watch for catalog regen conflicts |

**Next contributor move:** Sync fork from `orthogonalhq/nous-core`, confirm OpenRouter appears on updated integration tip, and pick a scoped follow-up (e.g. a live key-validation smoke test doc, or helping #413 once maintainer scopes it) rather than expanding the merged leaf without an issue.

---

## Timeline

| Date | Event |
|------|-------|
| 2026-06 (initial) | PR #410 opened — OpenRouter leaf + model discovery fix |
| 2026-06-28 | Maintainer review round 1 — three change requests |
| 2026-06-28 | Contributor acknowledged; review fixes implemented |
| 2026-06-30 | PR #410 merged by `@atlamors` into `feat/contributor-friendly-inference-provider-surface` |
| 2026-06-30 | Maintainer confirmed all PR-local items addressed; merge churn resolved maintainer-side |
