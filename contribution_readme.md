# Contribution 2: Adapter: Alibaba Qwen / DashScope Model Provider

**Contribution Number:** 2  
**Student:** Paul Piotrowski  
**Issue:** https://github.com/orthogonalhq/nous-core/issues/315  
**Fork:** https://github.com/PaulPio/nous-core  
**Status:** Phase II — Complete (reproduction executed + implementation plan finalized; no code, no PR)  
**Working branch:** [`feat/dashscope-provider-leaf`](https://github.com/PaulPio/nous-core/tree/feat/dashscope-provider-leaf) (cut from `upstream/feat/contributor-friendly-inference-provider-surface` @ `9604e213`)  
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

Today, attempting to use hosted Qwen requires manual workarounds (custom OpenAI base URL hacks or unrelated aggregators). The provider codegen roster under `self/subcortex/providers/src/providers/` has no `dashscope/` directory, so generated catalogs in `provider-definitions.ts`, `provider-factories.ts`, and `provider-adapters.ts` omit the vendor entirely. Two adjacent-but-different paths exist and neither closes the gap: local Qwen via Ollama (different protocol, no DashScope keys), and the `qwen-code/` **CLI** leaf, which merely allowlists `DASHSCOPE_API_KEY`/`DASHSCOPE_BASE_URL` as environment passthrough to the external Qwen Code CLI subprocess (`providers/qwen-code/implementation.ts:83-84`) — it is not a cloud inference provider and creates no Settings entry, vault namespace, or model discovery.

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
- [ ] Default endpoint targets international compatible-mode **base without the `/v1` suffix** (`https://dashscope-intl.aliyuncs.com/compatible-mode`) — the shared provider appends `/v1/...` itself (see doubled-`/v1` finding under Investigative Depth); document regional overrides.
- [ ] Capabilities declare only verified features (`streaming`, `modelListing`); omit `nativeToolUse` until #390.
- [ ] Leaf unit test + updated aggregate roster tests; `check:generated` clean.
- [ ] Manual smoke: provider visible in API Keys, valid key connects, model picker shows catalog or default model, chat completion succeeds.

---

## Reproduction Process

### Environment Setup

**Setup approach:** README/CONTRIBUTING instructions (the repo has no dev container). I followed `CONTRIBUTING.md` → Prerequisites (Node 22+, pnpm 10 via corepack, `pnpm install && pnpm build && pnpm test`) and cross-checked the canonical build/test commands against the CI config (`.github/workflows/ci-gate.yml` — typecheck, lint, test, build across Ubuntu/macOS/Windows), which is how I confirmed my 6 local test failures don't reproduce on CI's Linux runners (line-ending difference, detailed below). Repo-specific sharp edges (Electron `ELECTRON_RUN_AS_NODE` wrapper, better-sqlite3 build tools) are documented in the README's "Known Sharp Edges" and held true.

**Executed 2026-07-15 (Phase II), Windows 11, Node 22, pnpm 10.** Local dev was already proven from contribution 1 (OpenRouter #410); this cycle re-synced the integration branch and re-verified the provider-package baseline.

| Step | Command / action | Result |
|------|------------------|--------|
| Fork | https://github.com/PaulPio/nous-core | Active since #306 |
| Sync + branch | `git fetch upstream && git checkout -b feat/dashscope-provider-leaf upstream/feat/contributor-friendly-inference-provider-surface` | Branch tip `9604e213` (merge of xAI provider PR #424); pushed to fork |
| Install | `pnpm install` | OK (better-sqlite3 compiles — VS C++ workload already installed from contribution 1) |
| Generated catalogs | `pnpm --filter @nous/subcortex-providers run check:generated` | ✅ clean — catalogs in sync with leaves |
| Focused baseline | `pnpm test self/subcortex/providers` | 520 passed / **6 failed (all pre-existing)** / 4 skipped — see below |
| Dev UI | `pnpm dev:web` (port 4317) | Server starts, `Ready in 2.9s` |

Full gate (`pnpm typecheck && pnpm lint && pnpm test && pnpm build`) is deferred to Phase III alongside implementation.

**Challenges actually hit and how each was resolved:**

- **Challenge: 6 test failures on a supposedly clean checkout** — first run of `pnpm test self/subcortex/providers` failed 6 tests, which initially looked like a broken integration branch.
  **Resolution:** verified `git status` was clean (zero changes → failures can't be mine), then root-caused each class instead of guessing:
  - 2× `src/__tests__/provider-codegen.test.ts` — **Windows CRLF artifact**: read the failing assertion (`source.startsWith('// @generated …do not edit by hand.\n')`, lines 63-64) and checked the checkout encoding with `git ls-files --eol` → `i/lf w/crlf` under `core.autocrlf=true`. The file content is correct; only the line terminator differs from the LF-anchored assertion. Confirmed CI (Linux, LF) passes these. Resolved by recording it as a known local-environment baseline rather than "fixing" tests I don't own.
  - 3× `src/__tests__/providers/qwen-code.test.ts` — ran the file in isolation and read the assertion diffs (`spawn_error` vs `timeout`): Windows lacks POSIX SIGTERM semantics the live-runner tests assume. Same resolution: characterized and baselined.
  - 1× `src/__tests__/provider-pipeline-integration.test.ts` — env-var credential construction case, same environment class.
  Net: **flagged, not fixed** (repo guidance: report pre-existing breaks, don't scope-creep), and Phase III will diff its test results against this recorded baseline of exactly 6.
- **Challenge: finding the Settings → API Keys surface for UI reproduction** — `pnpm dev:web` starts clean (`Ready in 2.9s`) but there is no `/settings` route (404) and no Settings menu item in the web shell.
  **Resolution:** inspected the app router source (`self/apps/web/app/(shell)/` → `chat`, `config`, `mao`, `projects`, `traces`) and realized the stronger point: the Settings surface is *registry-driven* — it renders the generated `PROVIDER_DEFINITIONS` catalog, so catalog-level evidence is authoritative and a UI walk adds nothing. Documented that reasoning in Steps to Reproduce instead of a screenshot.
- **Regional endpoints (design input for Phase III):** DashScope uses `dashscope-intl.aliyuncs.com` (international) vs `dashscope.aliyuncs.com` (China) plus workspace-scoped hosts. The leaf `defaultEndpoint` picks the documented intl default; other regions override endpoint in provider config.
- **`/v1/models` availability:** some DashScope routes 404 on `/v1/models` while `/v1/chat/completions` works. Live probe with a real `DASHSCOPE_API_KEY` is deferred to Phase III — it decides whether the leaf needs a `healthCheckEndpoint` (OpenRouter used `/v1/key` for this in #410).
- **`OPENAI_API_KEY` leak (#413):** shared `ChatCompletionsProvider` can fall back to OpenAI's env var — the DashScope factory must fail closed, same pattern I shipped for OpenRouter.

### Steps to Reproduce (missing provider)

Executed 2026-07-15 on `feat/dashscope-provider-leaf` @ `9604e213` (clean tree). This is a *missing capability*, and the Settings/API-Keys surface is **registry-driven** — it renders whatever the generated `PROVIDER_DEFINITIONS` catalog contains. Absence from the catalog is therefore authoritative absence from the product; no UI walk is required to prove it.

**Expected behavior:** a user holding a `DASHSCOPE_API_KEY` can select DashScope in Settings → API Keys, store the key (vault namespace `dashscope`), discover Qwen models (or fall back to `qwen-plus`), and route chat/stream turns through hosted Qwen — exactly as Groq/OpenRouter users can today.

**Actual behavior:** no `dashscope/` leaf exists, so the generated catalogs contain no DashScope entry — there is no Settings field, no vault namespace, no factory that resolves `DASHSCOPE_API_KEY`, no model discovery, and no routing path. The env var is inert for inference (its only reference in the codebase is the `qwen-code` CLI subprocess allowlist). Steps 2–6 below demonstrate this observably.

1. Clone and check out the integration branch:
   ```bash
   git clone https://github.com/PaulPio/nous-core.git && cd nous-core
   git remote add upstream https://github.com/orthogonalhq/nous-core.git
   git fetch upstream
   git checkout -b feat/dashscope-provider-leaf upstream/feat/contributor-friendly-inference-provider-surface
   pnpm install
   ```
2. List the certified provider roster:
   ```bash
   ls self/subcortex/providers/src/providers/
   ```
   **Observed:** 18 vendor directories (`anthropic`, `codex-cli`, `deepinfra`, `gemini`, `github-copilot-cli`, `groq`, `huggingface-tgi`, `llama-cpp`, `mistral`, `moonshot`, `ollama`, `openai`, `openclaw`, `openrouter`, `perplexity`, `qwen-code`, `vllm`, `xai`) — **no `dashscope/`**. OpenRouter (#410) is present, confirming the branch includes prior provider merges.
3. Confirm the generated catalog omits the vendor:
   ```bash
   grep -c "dashscope" self/subcortex/providers/src/provider-definitions.ts   # → 0
   grep -rn -i "dashscope" self/subcortex/providers/src/
   ```
   **Observed:** the only hits are in the **`qwen-code` CLI leaf** (`providers/qwen-code/implementation.ts:83-84` allowlists `DASHSCOPE_API_KEY`/`DASHSCOPE_BASE_URL` as env *passthrough* to the external Qwen Code CLI subprocess, plus one doc-string in its `definition.ts`). No leaf declares `auth.envVar: 'DASHSCOPE_API_KEY'` — Nous can hand a DashScope key to an external CLI tool but has **no first-class DashScope cloud provider**: no Settings key field, no vault namespace, no model discovery, no chat-completions routing.
4. Confirm catalogs are fresh (so the absence isn't a stale-codegen illusion):
   ```bash
   pnpm --filter @nous/subcortex-providers run check:generated   # → exits 0, no drift
   ```
5. Run the provider test suite as a baseline:
   ```bash
   pnpm test self/subcortex/providers
   ```
   **Observed:** 520 passed / 6 failed / 4 skipped. All 6 failures are pre-existing Windows-environment artifacts on the untouched branch tip (CRLF header assertions in `provider-codegen.test.ts`, POSIX signal semantics in `qwen-code` live-runner tests, one pipeline-integration env case) — detailed under Environment Setup. **No test anywhere expects a `dashscope` roster entry**, confirming the vendor is absent from every aggregate roster (`provider-definitions.test.ts`, `provider-pipeline-integration.test.ts`, `adapter-resolver.test.ts`, …).
6. (Corollary, not separately run) Because Settings is generated from the catalog, a `DASHSCOPE_API_KEY` in the environment is inert for inference: no factory resolves it, no Settings entry stores it. The env var only reaches the sandboxed `qwen-code` CLI subprocess via its allowlist.
7. External control (deferred to Phase III with a real key): `curl https://dashscope-intl.aliyuncs.com/compatible-mode/v1/chat/completions` with Bearer auth succeeds per Alibaba's docs — the API exists; Nous simply has no leaf to route to it. This probe also settles `/v1/models` auth behavior for the `healthCheckEndpoint` decision.

### Reproduction Evidence

- **Branch link:** [`feat/dashscope-provider-leaf`](https://github.com/PaulPio/nous-core/tree/feat/dashscope-provider-leaf) on my fork — cut from `upstream/feat/contributor-friendly-inference-provider-surface` at `9604e213` and pushed 2026-07-15. Intentionally code-free for Phase II; Phase III commits land here.
- **Issue link:** [#315](https://github.com/orthogonalhq/nous-core/issues/315) — assigned to me, no linked PR.
- **My comment on issue:** [2026-07-05](https://github.com/orthogonalhq/nous-core/issues/315#issuecomment-4887810460); **maintainer reply** confirmed target branch `feat/contributor-friendly-inference-provider-surface`, that the provider surface is unchanged since #410 (current docs apply: quickstart / provider-leaf-anatomy / testing-checklist), and listed the open adapter-surface issues (#390, #391, #405, #408, #409, #413, #414) as planned-change context.
- **Codebase evidence (executed above):** 18-vendor roster with no `dashscope/`; generated catalogs omit the vendor (`check:generated` clean, so not a stale-codegen artifact); the only `DASHSCOPE_API_KEY` reference in `self/` is the `qwen-code` CLI env-passthrough allowlist — not a provider.
- **Baseline evidence:** provider suite 520 passed / 6 pre-existing environment failures on the clean branch tip (characterized under Environment Setup).
- **Dating the gap (git history):** `git log --all --oneline -- self/subcortex/providers/src/providers/dashscope/` returns **nothing** — the leaf has never existed in any branch's history; this is a capability gap since inception, not a regression to bisect. The surrounding timeline, from `git log --diff-filter=A`:
  - `58776ddf` 2026-06-10 — `refactor(providers): generate public catalogs directly` — the maintainer refactor that created today's leaf/codegen surface (the one I waited on during #306);
  - `d47ee852` 2026-06-21 — `qwen-code` **CLI** leaf added (source of the only `DASHSCOPE_API_KEY` mention — passthrough, not a provider);
  - `3f980e89` 2026-06-22 — `feat(providers): add OpenRouter provider leaf` (my #410 — the direct structural precedent, merged 8 days after the surface stabilized).
  Every cloud leaf added since the refactor follows the identical 4-file + codegen pattern, which is strong evidence the same pattern is the correct fix here.
- **My findings:** This is a *missing capability*, not a runtime regression. Reproduction = "vendor absent from the registry-driven catalog," which is authoritative for every downstream surface (Settings, vault, discovery, routing). Implementation is additive leaf work aligned with maintainer docs and the Groq/OpenRouter precedents.

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
- `defaultEndpoint: 'https://dashscope-intl.aliyuncs.com/compatible-mode'` — **deliberately without Alibaba's `/v1` suffix**: `ChatCompletionsProvider` joins `endpoint + '/v1/chat/completions'` (`protocols/openai-api/provider.ts:92`), so the base with `/v1` would double to `/compatible-mode/v1/v1/...` — the exact bug xAI shipped and fixed in `a4dc1950` (2026-07-14). Joined URLs resolve to Alibaba's documented `.../compatible-mode/v1/chat/completions` and `.../compatible-mode/v1/models`. (International default; document China alternate `https://dashscope.aliyuncs.com/compatible-mode`.)
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
- **Official provider-adapter docs** (in-repo at `docs/content/docs/development/provider-adapters/`, confirmed current by maintainer) — `quickstart.mdx` (leaf contract + codegen flow), `provider-leaf-anatomy.mdx` (file responsibilities, keys/protocols), `testing-checklist.mdx` (acceptance checklist + suggested commands).
- **Open adapter-surface issues as constraints** (maintainer-listed): #390 (omit `nativeToolUse` until the tool-use bridge lands), #413 (fail-closed factory — no `OPENAI_API_KEY` fallback), #391/#405/#408/#409/#414 (planned surface refinements — leaf must stay within the current contract, no anticipatory changes).

**Investigative Depth (stretch):** since this issue *adds* a provider rather than fixes a bug, the investigation mines the git history of the other recently-added leaves for mistakes to avoid and patterns to copy — treating each prior provider PR as a recorded experiment.

1. **Doubled-`/v1` trap — caught in the plan before writing code.** `git log` on the newest leaf (xAI, merged via PR #424 into the branch tip this plan sits on) surfaces commit `a4dc1950` (2026-07-14, *one day before this plan*): `fix(subcortex-providers): correct xAI endpoint to avoid doubled /v1 path`, a one-line diff `-'https://api.x.ai/v1'` → `+'https://api.x.ai'`. Root mechanics, verified in source: `ChatCompletionsProvider` builds request URLs as `endpoint.replace(/\/$/, '') + '/v1/chat/completions'` (`protocols/openai-api/provider.ts:24,92`), so any base ending in `/v1` doubles the segment. Cross-checking the whole roster confirms the invariant — every chat-completions leaf declares a base **without** `/v1` (`https://openrouter.ai/api`, `https://api.groq.com/openai`, `https://api.x.ai`, `https://api.moonshot.ai`). **Consequence for DashScope:** Alibaba documents its OpenAI-compatible base as `.../compatible-mode/v1`, so naively copying their docs reproduces xAI's bug. The plan's `defaultEndpoint` is therefore `https://dashscope-intl.aliyuncs.com/compatible-mode`, which the shared provider joins to exactly Alibaba's documented full URLs. A definition test will pin `defaultEndpoint` not ending in `/v1`.
2. **Closest vendor analog (beyond my own template):** Moonshot Kimi (`providers/moonshot/`, added 2026-06) is the nearest *business* analog — Chinese AI cloud, Bearer auth, OpenAI-compatible endpoint — while OpenRouter remains the *structural* template. Comparing them exposes a roster inconsistency worth not repeating: Moonshot declares `modelListEndpoint` but omits `modelListFormat` and `capabilities.modelListing`, which the official testing-checklist says must travel together for discovery to work. DashScope declares all three (OpenRouter/Groq pattern).
3. **Capability honesty divergence:** xAI and Moonshot ship `nativeToolUse: true` while Groq/OpenRouter deliberately omit it pending the #390 tool-use bridge. The maintainer's guidance on #315 (and my #410 review) says omit — DashScope follows the conservative half of the roster; noted so the PR description can preempt the review question.
4. **Edge cases identified proactively** (each mapped to a planned test or a documented decision): regional bases (intl default, China override — decision documented); `/v1/models` 404 on some DashScope hosts (live probe deferred to Phase III; decides `healthCheckEndpoint` — OpenRouter precedent `/v1/key`); `OPENAI_API_KEY`/`OPENAI_API_BASE` fallbacks inside the shared provider (`provider.ts:69,71` — neutralized by the fail-closed factory test + registry always passing `config.endpoint`); trailing-slash bases (already normalized by `provider.ts:92` — covered by existing protocol tests); Windows CRLF codegen churn and parallel-provider-PR roster ordering (both observed live in this Phase's baseline — re-run `generate:providers` before commit, expect catalog merge churn at PR time).

**Plan:**

1. Branch `feat/dashscope-provider-leaf` from `upstream/feat/contributor-friendly-inference-provider-surface`.
2. Create `providers/dashscope/{definition,adapter,provider,index}.ts`.
3. Run `generate:providers`; verify `check:generated`.
4. Add `__tests__/providers/dashscope.test.ts` (metadata, schema hydration, factory auth, no `OPENAI_API_KEY` fallback).
5. Update aggregate tests: `provider-definitions.test.ts`, `provider-definition-types.test.ts`, `provider-pipeline-integration.test.ts`, `provider-codegen.test.ts`, `adapter-resolver.test.ts`, `provider-registry.test.ts`.
6. If DashScope `/v1/models` response shape differs, minimal fix in `provider-model-discovery.ts` + shared-server test (OpenRouter precedent).
7. Manual smoke with real `DASHSCOPE_API_KEY` on `pnpm dev:web`.
8. Open PR targeting `feat/contributor-friendly-inference-provider-surface`.

**Implement:** Branch ready — [`feat/dashscope-provider-leaf`](https://github.com/PaulPio/nous-core/tree/feat/dashscope-provider-leaf) (code lands in Phase III; commits will appear on this link).

**Review:**

- [ ] No hand-authored `wellKnownProviderId`
- [ ] Generated catalogs not manually edited
- [ ] No `@nous/shared` interface changes
- [ ] No unrelated fixes bundled
- [ ] `contribution_readme.md` stays local (not committed)

**Evaluate:** Testing-checklist suggested commands first —

```bash
pnpm --filter @nous/subcortex-providers run check:generated
pnpm --filter @nous/subcortex-providers run typecheck
pnpm --filter @nous/subcortex-providers exec vitest run src/__tests__/provider-codegen.test.ts src/__tests__/public-exports.test.ts src/__tests__/provider-definitions src/__tests__/adapter-resolver.test.ts src/__tests__/provider-pipeline-integration.test.ts --config vitest.config.ts
```

then the repo gate (`pnpm typecheck && pnpm lint && pnpm test self/subcortex/providers && pnpm build`), optional `pnpm test provider-model-discovery` if shared-server discovery is touched, and a manual Settings smoke on `pnpm dev:web`. Baseline caveat: compare against the 6 pre-existing Windows-environment failures recorded in Phase II — the leaf must add zero new failures.

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

### Week 2 Progress (Phase II — 2026-07-15)

- Cut and pushed [`feat/dashscope-provider-leaf`](https://github.com/PaulPio/nous-core/tree/feat/dashscope-provider-leaf) from `upstream/feat/contributor-friendly-inference-provider-surface` @ `9604e213`.
- Executed the full reproduction: roster listing (18 vendors, no `dashscope/`), generated-catalog grep (0 hits), `check:generated` clean, provider test baseline (520 passed / 6 pre-existing Windows-environment failures, characterized — CRLF codegen assertions, `qwen-code` live-runner signal semantics, one pipeline env case).
- Key discovery sharpening the repro: the only `DASHSCOPE_API_KEY` in the codebase is the `qwen-code` CLI env-passthrough allowlist — confirms "no first-class DashScope provider" precisely.
- Maintainer confirmed on #315: target branch unchanged, provider surface stable since #410, docs current, open adapter-surface issues listed — folded into plan constraints.
- Finalized the UMPIRE implementation plan (below); grounded Evaluate in the official testing-checklist commands.
- **Next (Phase III):** live endpoint probe with real `DASHSCOPE_API_KEY` (decides `healthCheckEndpoint`), implement the four leaf files, codegen, tests, PR.

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

**From contribution 2 Phase II (#315):**
- Reproducing a *missing capability* is different from reproducing a bug: the authoritative evidence is the registry-driven catalog (roster + generated files + `check:generated`), not a UI walk — the Settings surface renders whatever the catalog contains.
- Learned to separate pre-existing baseline failures from contribution-caused ones: traced 2 codegen test failures to Windows CRLF checkout (`core.autocrlf=true` vs LF-anchored `startsWith` assertion) rather than assuming broken code — recorded as baseline so Phase III diffs against it.
- Grep before you claim: "no DASHSCOPE_API_KEY anywhere" would have been wrong — the `qwen-code` CLI leaf allowlists it as env passthrough. Precision here made the reproduction claim stronger, not weaker.

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
- `CLAUDE.md`, `CONTRIBUTING.md`, in-repo provider-adapter docs (`docs/content/docs/development/provider-adapters/` — quickstart, provider-leaf-anatomy, testing-checklist, schemas-abi-reference, what-not-to-edit)
