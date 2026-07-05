# Contribution 2: Adapter: Alibaba Qwen / DashScope Model Provider

**Contribution Number:** 2  
**Student:** Paul Piotrowski  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/315  
**Fork:** https://github.com/PaulPio/nous-core  
**Status:** Phase I — In Progress  
**Prior contribution:** [#306 OpenRouter provider](https://github.com/orthogonalhq/nous-core/issues/306) → [PR #410 merged 2026-06-30](https://github.com/orthogonalhq/nous-core/pull/410)

---

## Why I Chose This Issue

This is my **second provider-leaf contribution** to Nous. For contribution 1 I implemented and merged the OpenRouter provider ([#306](https://github.com/orthogonalhq/nous-core/issues/306), [PR #410](https://github.com/orthogonalhq/nous-core/pull/410)) on `feat/contributor-friendly-inference-provider-surface` — including the fail-closed factory, `healthCheckEndpoint` for key validation, and a shared-server model-discovery compatibility fix. Maintainer `@atlamors` unblocked that work by refactoring the adapter surface after I claimed the issue ([comment thread, 2026-06-07](https://github.com/orthogonalhq/nous-core/issues/306#issuecomment-4644415563)); the merged leaf is now live at `self/subcortex/providers/src/providers/openrouter/`. Issue [#315](https://github.com/orthogonalhq/nous-core/issues/315) is the natural follow-on: same `adapter` / `good first issue` shape, same OpenAI-compatible protocol, same integration branch — but a different vendor with its own endpoint and auth quirks.

Nous already supports local Qwen through Ollama (e.g. `ollama:qwen2.5:7b`), but there is no first-class path to Alibaba Cloud's **DashScope** API — hosted Qwen for teams that want managed inference without running weights locally. DashScope closes a real product gap (direct Qwen cloud access vs. routing through an aggregator like OpenRouter), and I already have the repo context: pnpm monorepo tooling, `ProviderDefinitionLeaf` + `generate:providers`, aggregate test rosters, and the maintainer review loop from #410.

**Skill match:** I have already shipped one certified leaf end-to-end in this codebase. DashScope reuses the same `ChatCompletionsProvider` primitive maintainer `@atlamors` described on both issues — metadata + factory, not a new adapter class. My primary template is **my own OpenRouter leaf**, with Groq as the minimal baseline.

**Learning goals for contribution 2:** (1) apply the leaf pattern to a second vendor, validating the workflow is repeatable; (2) handle DashScope-specific complexity OpenRouter did not have — multi-region compatible-mode bases (`dashscope.aliyuncs.com` vs `dashscope-intl.aliyuncs.com`), possible `/v1/models` gaps on some hosts, and choosing `healthCheckEndpoint` vs model-list for key proof; (3) deliver a tighter Phase I plan and issue comment up front, using lessons from the #306 thread where I waited on the maintainer refactor before starting. I chose this over unrelated bug fixes because it extends a capability area I have already proven in review and merge.

---

## Understanding the Issue

### Problem Description

Nous's subcortex provider registry has certified leaves for Anthropic, OpenAI, Groq, OpenRouter, Ollama, and others under `self/subcortex/providers/src/providers/`, but **no leaf for Alibaba DashScope (Qwen cloud)**. Users with a `DASHSCOPE_API_KEY` cannot add DashScope in Settings → API Keys, cannot pick Qwen cloud models in the model picker, and cannot route agent turns through hosted Qwen inference. The issue is labeled `good first issue` / `adapter` and explicitly scopes work to the current provider leaf contract (not the superseded `IModelProvider` monolith path).

DashScope exposes an [OpenAI-compatible Chat Completions API](https://www.alibabacloud.com/help/en/model-studio/compatibility-of-openai-with-dashscope) at paths like `https://dashscope-intl.aliyuncs.com/compatible-mode/v1/chat/completions` (international) or `https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions` (China), authenticated with `Authorization: Bearer $DASHSCOPE_API_KEY`. That means inference reuses Nous's existing `ChatCompletionsProvider` in `self/subcortex/providers/src/protocols/openai-api/` — the leaf only supplies vendor metadata, auth, endpoints, and a safe factory.

### Expected Behavior

After the provider leaf lands (target branch: `feat/contributor-friendly-inference-provider-surface` per maintainer update 2026-06-18):

1. **DashScope / Qwen** appears in Settings → API Keys without custom UI code (registry-driven from the leaf definition).
2. Users store a `DASHSCOPE_API_KEY`; bootstrap injects it into `process.env` per `auth.envVar`.
3. The model picker discovers Qwen models via `modelListEndpoint` + `modelListFormat: 'openai-models'`, or falls back to a sensible `defaultModelId` (e.g. `qwen-plus`) if listing is unavailable for a given endpoint variant.
4. Agent turns stream completions from DashScope through the shared chat-completions protocol.
5. `pnpm --filter @nous/subcortex-providers run check:generated` passes; provider unit and aggregate tests include the new vendor roster entry.
6. Built-in provider ID is **derived from `vendorKey`** — no hand-authored `wellKnownProviderId`.

### Current Behavior

Today, attempting to use hosted Qwen requires manual workarounds (custom OpenAI base URL hacks or unrelated aggregators). The provider codegen roster under `self/subcortex/providers/src/providers/` has no `dashscope/` (or `qwen/`) directory, so generated catalogs in `provider-definitions.ts`, `provider-factories.ts`, and `provider-adapters.ts` omit the vendor entirely. Local Qwen via Ollama works, but that is a different protocol (`ollama`) and does not accept DashScope API keys.

### Affected Components

| Area | Path | Role |
|------|------|------|
| **New provider leaf** | `self/subcortex/providers/src/providers/dashscope/` | `definition.ts`, `adapter.ts`, `provider.ts`, `index.ts` |
| **Shared OpenAI protocol** | `self/subcortex/providers/src/protocols/openai-api/` | `ChatCompletionsProvider` + `chatCompletionsAdapter` (reuse, not fork) |
| **Codegen output** | `provider-definitions.ts`, `provider-factories.ts`, `provider-adapters.ts` | Regenerated via `generate:providers` — never hand-edited |
| **Provider identity** | `self/subcortex/providers/src/provider-identity.ts` | Derives built-in ID from `vendorKey` |
| **Model discovery** | `self/apps/shared-server/src/provider-model-discovery.ts` | Parses `/v1/models` responses (`openai-models` format); may need DashScope-shaped test if fields differ |
| **Key validation** | Same file — `testProviderApiKey()` | Uses `healthCheckEndpoint ?? modelListEndpoint`; must confirm DashScope auth behavior |
| **Reference leaves** | `providers/openrouter/` _(my merged #410 work)_, `providers/groq/` | OpenRouter = primary template (fail-closed factory, `healthCheckEndpoint`, discovery tests); Groq = minimal baseline |
| **Aggregate tests** | `self/subcortex/providers/src/__tests__/providers/*.test.ts`, `provider-definitions.test.ts`, `provider-pipeline-integration.test.ts`, etc. | Roster updates when adding a vendor |

**Maintainer guidance (issue comments):**

- [@atlamors, 2026-04-16](https://github.com/orthogonalhq/nous-core/issues/315#issuecomment-4262815139): DashScope is OpenAI-compatible; ship as config on `ChatCompletionsProvider`. Watch for hard-coded `OPENAI_API_KEY` fallback (#324 / #413) — use fail-closed leaf factory until shared cleanup lands.
- [@atlamors, 2026-06-12](https://github.com/orthogonalhq/nous-core/issues/315#issuecomment-4686350451): Implement certified leaf; do not hand-edit catalogs; confirm integration branch before PR; reuse `protocols/openai-api` where API is actually compatible.
- **Issue body (2026-06-18):** Integration target is `feat/contributor-friendly-inference-provider-surface`; docs at [provider adapter quickstart](https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart).

**Concrete acceptance criteria (“done” looks like):**

- [ ] `dashscope` leaf passes `ProviderDefinitionSchema` hydration (no manual UUID).
- [ ] `DASHSCOPE_API_KEY` is the sole credential source at the factory boundary (no `OPENAI_API_KEY` fallback).
- [ ] Default endpoint targets international compatible-mode (`https://dashscope-intl.aliyuncs.com/compatible-mode/v1`) unless research shows a better single default; document regional overrides.
- [ ] Capabilities declare only verified features (`streaming`, `modelListing`); omit `nativeToolUse` until #390.
- [ ] Leaf unit test + updated aggregate roster tests; `check:generated` clean.
- [ ] Manual smoke: provider visible in API Keys, valid key connects, model picker shows catalog or default model, chat completion succeeds.

---

## Reproduction Process

### Environment Setup

**Goal for Phase I:** Confirm the *absence* of DashScope as a provider. Local dev is already proven from contribution 1 (OpenRouter #410): fork, `pnpm install`, integration branch, and `pnpm dev:web` were all exercised during that cycle.

| Step | Command / action | Notes |
|------|------------------|-------|
| Fork (done) | https://github.com/PaulPio/nous-core | Created for #306; still active |
| Sync integration branch | `git fetch upstream && git checkout feat/contributor-friendly-inference-provider-surface && git pull upstream feat/contributor-friendly-inference-provider-surface` | Same target branch as #306/#410; pull to pick up merged OpenRouter leaf |
| Verify baseline | `pnpm typecheck && pnpm lint && pnpm test && pnpm build` | Re-run after sync; flag pre-existing breaks without scope-creeping |
| Dev UI | `pnpm dev:web` (port 4317) | Settings → API Keys — OpenRouter **is** listed (from #410); DashScope is **not** |

**Challenges anticipated:**

- **Regional endpoints:** DashScope uses different base URLs for China (`dashscope.aliyuncs.com`) vs international (`dashscope-intl.aliyuncs.com`) and newer workspace-scoped URLs (`{WorkspaceId}.{region}.maas.aliyuncs.com`). The leaf `defaultEndpoint` must pick a documented default; users in other regions may override endpoint in config.
- **`/v1/models` availability:** Some DashScope routes (e.g. Coding Plan hosts) return 404 on `/v1/models` while `/v1/chat/completions` still works ([hermes-agent#12220](https://github.com/NousResearch/hermes-agent/issues/12220)). Plan: verify against compatible-mode endpoint; if listing fails, rely on `defaultModelId` and document limitation; add `healthCheckEndpoint` if model list is public (OpenRouter pattern).
- **`OPENAI_API_KEY` leak:** Shared `ChatCompletionsProvider` can fall back to OpenAI's env var — I fixed this at the OpenRouter leaf boundary in #410; DashScope must use the same fail-closed factory pattern I already shipped.

### Steps to Reproduce (missing provider)

1. Check out `feat/contributor-friendly-inference-provider-surface` and run `pnpm dev:web`.
2. Open Settings → API Keys → provider dropdown.
3. **Observed:** Groq, **OpenRouter** (my #410 contribution), OpenAI, Anthropic, etc. appear; **DashScope / Qwen cloud does not**.
4. Search codebase: `ls self/subcortex/providers/src/providers/` — **no `dashscope/` directory**.
5. Optional external control: with a valid `DASHSCOPE_API_KEY`, `curl` against `https://dashscope-intl.aliyuncs.com/compatible-mode/v1/chat/completions` succeeds — proving the API works but Nous has no leaf to route to it.

### Reproduction Evidence

- **Issue link:** [#315](https://github.com/orthogonalhq/nous-core/issues/315) — open, unassigned, no linked PR.
- **My comment on issue:** [2026-07-05](https://github.com/orthogonalhq/nous-core/issues/315#issuecomment-4887810460) — introduced interest; awaiting maintainer confirmation of target branch.
- **Codebase evidence:** Provider roster ends at `openrouter/`, `groq/`, `anthropic/`, etc.; grep for `dashscope` in `self/subcortex/providers/` returns no provider leaf.
- **Commit showing reproduction:** _(Phase II — will link first commit on `feat/dashscope-provider-leaf` that adds a failing roster test or documents absence.)_
- **My findings:** This is a *missing capability*, not a runtime regression. Reproduction is "provider absent from registry + Settings UI." Implementation is additive leaf work aligned with maintainer docs and Groq/OpenRouter precedents.

---

## Solution Approach

### Analysis

**Root cause:** The provider registry is leaf-driven. Each vendor is a directory under `src/providers/<vendor>/` discovered by codegen. DashScope was never added, so catalogs, factories, Settings, and model discovery have no entry for it.

**Why ChatCompletionsProvider fits:** Maintainer confirmed DashScope's compatible-mode API matches OpenAI Chat Completions (`POST .../v1/chat/completions`, Bearer auth, streaming). No new protocol class is required — only metadata and a safe factory.

**Risk areas:**

1. **Credential isolation** — must fail closed at `provider.ts` (OpenRouter pattern).
2. **Endpoint / model-list quirks** — validate `/v1/models` on compatible-mode; separate key-validation endpoint if catalog is public.
3. **Capability honesty** — do not set `nativeToolUse: true` until #390 bridge exists.

### Proposed Solution

Add a certified **DashScope provider leaf** at `self/subcortex/providers/src/providers/dashscope/` mirroring **my merged OpenRouter leaf** (`providers/openrouter/`) for structure, auth safety, and test coverage — with Groq as the minimal fallback if DashScope needs fewer fields. The leaf declares:

- `vendorKey: 'dashscope'`, `displayName: 'DashScope (Qwen)'`
- `protocol: 'chat-completions'`, `adapterKey: 'chat-completions'`
- `defaultEndpoint: 'https://dashscope-intl.aliyuncs.com/compatible-mode/v1'` (international default; document China alternate)
- `defaultModelId: 'qwen-plus'` (widely documented starter model)
- `auth.envVar: 'DASHSCOPE_API_KEY'`, Bearer header, vault namespace `dashscope`
- `modelListEndpoint: '/v1/models'`, `modelListFormat: 'openai-models'`
- `healthCheckEndpoint`: TBD after manual probe — add if `/v1/models` does not prove credentials
- Fail-closed `providerFactory.create()` resolving only `DASHSCOPE_API_KEY`

Then run `pnpm --filter @nous/subcortex-providers run generate:providers` and extend test rosters.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Nous lacks a DashScope/Qwen cloud provider. DashScope offers OpenAI-compatible chat completions; users need API-key settings, model selection, and inference routing without custom UI.

**Match:**

- **OpenRouter leaf** (`providers/openrouter/`) — **my #410 implementation**; primary template for fail-closed factory, `healthCheckEndpoint`, unit + aggregate tests, and optional `provider-model-discovery.ts` fix if DashScope's `/v1/models` shape diverges.
- **Groq leaf** (`providers/groq/`) — minimal OpenAI-compatible cloud metadata when DashScope needs no extra endpoints.
- **Prior review feedback from #410** — fail-closed credentials, separate key validation from public model catalogs, no process docs in repo.
- **Provider skill** (`.cursor/skills/nous-provider-leaf/`) — checklist distilled from contribution 1 + maintainer merge.

**Plan:**

1. Branch `feat/dashscope-provider-leaf` from `upstream/feat/contributor-friendly-inference-provider-surface`.
2. Create `providers/dashscope/{definition,adapter,provider,index}.ts`.
3. Run `generate:providers`; verify `check:generated`.
4. Add `__tests__/providers/dashscope.test.ts` (metadata, schema hydration, factory auth, no `OPENAI_API_KEY` fallback).
5. Update aggregate tests: `provider-definitions.test.ts`, `provider-definition-types.test.ts`, `provider-pipeline-integration.test.ts`, `provider-codegen.test.ts`, `adapter-resolver.test.ts`, `provider-registry.test.ts`.
6. If DashScope `/v1/models` response shape differs, minimal fix in `provider-model-discovery.ts` + shared-server test (OpenRouter precedent).
7. Manual smoke with real `DASHSCOPE_API_KEY` on `pnpm dev:web`.
8. Open PR targeting `feat/contributor-friendly-inference-provider-surface`.

**Implement:** _(Phase II — link branch and commits here.)_

**Review:**

- [ ] No hand-authored `wellKnownProviderId`
- [ ] Generated catalogs not manually edited
- [ ] No `@nous/shared` interface changes
- [ ] No unrelated fixes bundled
- [ ] `contribution_readme.md` stays local (not committed)

**Evaluate:** Full gate — `generate:providers`, `check:generated`, `pnpm typecheck`, `pnpm lint`, `pnpm test self/subcortex/providers`, optional `pnpm test provider-model-discovery`, `pnpm build`, manual Settings smoke.

---

## Testing Strategy

### Unit Tests

- [ ] **Definition metadata:** `vendorKey`, `displayName`, `protocol`, `defaultEndpoint`, `defaultModelId`, `DASHSCOPE_API_KEY`, Bearer auth, model list fields.
- [ ] **Schema hydration:** hydrated definition passes `ProviderDefinitionSchema`; no manual `wellKnownProviderId`.
- [ ] **Capabilities:** `streaming` + `modelListing` present; `nativeToolUse` absent.
- [ ] **Factory:** builds `ChatCompletionsProvider` with `deriveBuiltInProviderId('dashscope')`.
- [ ] **Auth fail-closed:** throws `NousError` / `PROVIDER_AUTH_FAILED` when key missing.
- [ ] **Regression:** factory does **not** use `OPENAI_API_KEY` when `DASHSCOPE_API_KEY` unset.

### Integration Tests

- [ ] `provider-codegen.test.ts` — leaf discovered; generated files in sync.
- [ ] `provider-definitions.test.ts` — roster includes `dashscope`.
- [ ] `provider-pipeline-integration.test.ts` — registry constructs `ChatCompletionsProvider` with `DASHSCOPE_API_KEY`.
- [ ] `provider-registry.test.ts` — routing uses DashScope env var, not OpenAI.
- [ ] _(If needed)_ `provider-model-discovery.test.ts` — DashScope-shaped `/v1/models` parse + key validation.

### Manual Testing

- [ ] DashScope appears in Settings → API Keys dropdown.
- [ ] Invalid key fails validation (once `healthCheckEndpoint` confirmed).
- [ ] Valid key stores and connects.
- [ ] Model picker lists Qwen models (or shows `qwen-plus` default).
- [ ] Send a chat turn; receive streamed Qwen response.

---

## Implementation Notes

### Week 1 Progress (Phase I)

- **Contribution 1 complete:** [#306](https://github.com/orthogonalhq/nous-core/issues/306) OpenRouter leaf merged via [#410](https://github.com/orthogonalhq/nous-core/pull/410) (2026-06-30); assigned on #306; maintainer refactor unblocked work per [2026-06-07 thread](https://github.com/orthogonalhq/nous-core/issues/306#issuecomment-4644415563).
- Selected [#315](https://github.com/orthogonalhq/nous-core/issues/315) as contribution 2 — same provider-leaf pattern, new vendor.
- Re-read maintainer updates on #315; compared DashScope compatible-mode docs against my OpenRouter `definition.ts` / `provider.ts`.
- Commented on #315 ([2026-07-05](https://github.com/orthogonalhq/nous-core/issues/315#issuecomment-4887810460)) requesting target-branch confirmation.
- Completed Phase I contribution README with scoped plan, affected files, and acceptance criteria.
- **Next:** Sync fork from integration branch (includes my OpenRouter leaf); cut `feat/dashscope-provider-leaf` and implement.

### Code Changes

- **Files to add:** `self/subcortex/providers/src/providers/dashscope/{definition,adapter,provider,index}.ts`; `__tests__/providers/dashscope.test.ts`.
- **Files to regenerate:** `provider-definitions.ts`, `provider-factories.ts`, `provider-adapters.ts`.
- **Files possibly touched:** aggregate provider tests; optionally `provider-model-discovery.ts` if parser edge case found.
- **Key commits:** _(Phase II+)_
- **Approach decisions:** Use `vendorKey: 'dashscope'` and `DASHSCOPE_API_KEY` to match Alibaba docs; international compatible-mode as default endpoint; copy fail-closed factory + health-check patterns directly from my OpenRouter leaf (#410).

---

## Pull Request

**PR Link:** _(not yet submitted)_

**PR Description (draft):**

> Adds a certified DashScope (Qwen cloud) provider leaf under `self/subcortex/providers/src/providers/dashscope/`. DashScope exposes an OpenAI Chat Completions-compatible API, so the leaf supplies vendor metadata and reuses `ChatCompletionsProvider`. Built-in provider ID derives from `vendorKey`; catalogs regenerated via `generate:providers`. DashScope appears in API Keys settings; model discovery uses `/v1/models` where supported.
>
> Closes #315.

**Maintainer Feedback:**

- _(pending)_

**Status:** Awaiting implementation (Phase II)

---

## Learnings & Reflections

### Technical Skills Gained

**From contribution 1 (#306 / #410):**
- End-to-end provider leaf: `definition.ts` → codegen → aggregate test roster updates → maintainer review iteration → merge.
- OpenAI-compatible integration pitfalls: strict `/v1/models` Zod parsing, public catalog vs authenticated key validation, `ChatCompletionsProvider` credential fallback (#413).
- Registry-driven surfacing — no app/UI edits needed for standard API-key providers.

**From contribution 2 Phase I (#315):**
- Researched DashScope multi-region compatible-mode endpoints and `/v1/models` availability gaps across endpoint variants (beyond what OpenRouter required).

### Challenges Overcome

- **Contribution 1 — waiting on maintainer refactor:** On #306, `@atlamors` prioritized the adapter-surface refactor after I claimed the issue ([2026-06-07](https://github.com/orthogonalhq/nous-core/issues/306#issuecomment-4644415563)); I held implementation until the leaf contract stabilized, then delivered #410 in one focused PR cycle with a fast review turnaround.
- **Contribution 2 — vendor-specific endpoints:** DashScope is not a single global URL like Groq or OpenRouter; choosing a default while documenting regional overrides is the main new design question for #315.

### What I'd Do Differently Next Time

- On #315 Phase I: include a concrete implementation sketch in the issue comment (proposed `vendorKey`, endpoint, env var) alongside this README — I did that on #306 only after claiming; doing both together speeds maintainer feedback.

---

## Resources Used

**This contribution (#315):**
- [Issue #315 — Adapter: Alibaba Qwen / DashScope Model Provider](https://github.com/orthogonalhq/nous-core/issues/315)
- [Alibaba Cloud — OpenAI compatibility with DashScope](https://www.alibabacloud.com/help/en/model-studio/compatibility-of-openai-with-dashscope)
- [Alibaba Cloud — First API call to Qwen](https://www.alibabacloud.com/help/en/model-studio/first-api-call-to-qwen)

**Prior contribution (#306) — direct precedent:**
- [Issue #306 — OpenRouter Model Provider](https://github.com/orthogonalhq/nous-core/issues/306) (closed, assigned to me)
- [Maintainer unblock thread](https://github.com/orthogonalhq/nous-core/issues/306#issuecomment-4644415563) — adapter-surface refactor before implementation
- [PR #410 — merged OpenRouter leaf](https://github.com/orthogonalhq/nous-core/pull/410) — my primary code reference

**Nous docs & repo:**
- [Provider adapter quickstart](https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart)
- [Provider leaf anatomy](https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy)
- Reference leaves: `self/subcortex/providers/src/providers/openrouter/` (authored), `providers/groq/` (minimal)
- `CLAUDE.md`, `CONTRIBUTING.md`, `.cursor/skills/nous-provider-leaf/SKILL.md`
