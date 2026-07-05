---
name: nous-provider-leaf
description: >-
  Add certified model provider leaves to Nous (nous-core) under
  self/subcortex/providers. Covers definition/adapter/provider/index scaffolding,
  codegen, discovery, key validation, tests, and PR checklist. Use when adding
  a new inference provider (OpenAI-compatible, Anthropic, Ollama, local), Groq/
  OpenRouter-style leaves, or when the user mentions provider integration,
  vendorKey, or generate:providers.
---

# Nous Provider Leaf

Add a **certified provider leaf** under `self/subcortex/providers/src/providers/<vendor>/`. The registry-driven surface means most API-key providers need **only the leaf** — Settings, model discovery, and vault injection are definition-driven.

Read [reference.md](reference.md) for file templates, test roster, and pitfall details.

## Before You Code

1. **Confirm protocol** — pick the closest existing leaf (table below).
2. **Branch** — cut from the maintainer integration branch if contributing upstream (e.g. `feat/contributor-friendly-inference-provider-surface`), not always `main`.
3. **Read contracts** — `self/shared/src/interfaces/` only if changing boundaries; leaves should not touch `IModelProvider` or `@nous/shared` for standard integrations.
4. **Do not edit generated files by hand** — `provider-definitions.ts`, `provider-factories.ts`, `provider-adapters.ts`.

## Protocol → Reference Leaf

| Protocol | When | Reference leaf | Shared implementation |
|----------|------|----------------|------------------------|
| `chat-completions` | OpenAI-compatible HTTP API | `providers/groq/` | `protocols/openai-api/provider.ts` |
| `chat-completions` + auth quirks | Aggregator / non-OpenAI-shaped `/v1/models` | `providers/openrouter/` | same + `healthCheckEndpoint`, fail-closed factory |
| `chat-completions` | Local OpenAI-compatible server | `providers/llama-cpp/` | same; `isLocal: true`, `auth.required: false` |
| `anthropic-messages` | Native Anthropic API | `providers/anthropic/` | `providers/anthropic/implementation.ts` |
| `ollama` | Ollama daemon | `providers/ollama/` | `providers/ollama/implementation.ts` |
| `agent-cli` | External CLI agent | `providers/codex-cli/` | `protocols/agent-cli/` |

**Default for new cloud APIs:** mirror **Groq** unless the vendor diverges (then check **OpenRouter** patterns).

## Leaf Checklist

Create `self/subcortex/providers/src/providers/<vendor>/` with **all four** files (codegen enforces this):

```
providers/<vendor>/
├── definition.ts   # PROVIDER_DEFINITION — metadata only
├── adapter.ts      # re-export protocol adapter
├── provider.ts     # providerFactory.create → protocol provider
└── index.ts        # barrel exports
```

Optional: `implementation.ts` when the protocol class lives in the leaf (Anthropic, Ollama, codex-cli).

### definition.ts rules

- `vendorKey`: lowercase kebab-case matching directory name (`groq`, `openrouter`, `llama-cpp`).
- **No hand-authored `wellKnownProviderId`** — derived from `vendorKey` via `provider-identity.ts`.
- `as const satisfies ProviderDefinitionLeaf`.
- Export as `providerDefinition` (codegen imports this name).
- Remote API-key providers need:
  - `auth.envVar`, `auth.vaultKeyNamespace`, `auth.header` (`bearer` or `raw`), `auth.required: true`, `auth.purpose: 'api_key'`
  - `modelListEndpoint` + `modelListFormat` (`openai-models` or `anthropic-models`) for model picker
- **Do not advertise `nativeToolUse`** until #390 bridge exists (omit the field; Groq/OpenRouter pattern).
- Capabilities: only declare what is verified (`streaming`, `modelListing`, etc.).

### provider.ts rules

- Export `providerFactory` with `vendorKey` and `create(config, options)`.
- **OpenAI-compatible cloud leaves:** if reusing `ChatCompletionsProvider`, avoid passing `undefined` apiKey — it falls back to `OPENAI_API_KEY` internally (#413). Either:
  - **Fail closed** (OpenRouter pattern): resolve only your `auth.envVar`, throw `NousError` with `PROVIDER_AUTH_FAILED`, then pass concrete `apiKey`.
  - **Local/no-auth** (llama-cpp): pass `apiKey: options?.apiKey ?? 'no-auth'`.
- OpenAI vendor leaf may pass `options?.apiKey` through (intentional OpenAI fallback).

### adapter.ts

Re-export the protocol adapter as `providerAdapter`:

```ts
export { chatCompletionsAdapter as providerAdapter } from '../../protocols/openai-api/adapter.js';
```

## Registry-Driven Surfacing (no app/UI edits for standard leaves)

From the definition alone, Nous automatically:

- Shows the provider in **Settings → API Keys** (`auth.header` + `envVar`)
- Loads stored keys into `process.env[envVar]` on bootstrap
- Discovers models via `modelListEndpoint` + `modelListFormat` (`provider-model-discovery.ts`)
- Tests keys via `healthCheckEndpoint ?? modelListEndpoint` (`testProviderApiKey`)

### Key validation vs model discovery

| Endpoint role | Field | Notes |
|---------------|-------|-------|
| Model catalog | `modelListEndpoint` | e.g. `/v1/models` |
| Key proof | `healthCheckEndpoint` | **Required** if model list is public or returns 200 without auth (OpenRouter: `/v1/key`) |

`testProviderApiKey` uses `healthCheckEndpoint` first; only falls back to `modelListEndpoint`.

### OpenAI-compatible model list parsing

If `/v1/models` omits `object` / `owned_by`, discovery already treats those as optional in `provider-model-discovery.ts`. If a new vendor omits other required fields, extend the Zod schema minimally and add a vendor-shaped test.

## Codegen & Build

```bash
pnpm --filter @nous/subcortex-providers run generate:providers
pnpm --filter @nous/subcortex-providers run check:generated   # CI gate
pnpm --filter @nous/subcortex-providers run build
```

Codegen discovers every directory under `src/providers/` matching `[a-z][a-z0-9]*(-[a-z0-9]+)*` with the four required files.

## Tests (required)

### 1. Leaf unit test — `src/__tests__/providers/<vendor>.test.ts`

- Definition metadata (endpoint, envVar, header, model list, health check)
- No hand-authored `wellKnownProviderId`; hydrated schema parse
- Capabilities (including `nativeToolUse` absent when appropriate)
- Factory constructs correct protocol class
- **Auth:** fail-closed or env resolution; regression that wrong env vars are not used

### 2. Update aggregate suites (roster changes when adding a vendor)

| Test file | What to update |
|-----------|----------------|
| `provider-definitions/provider-definitions.test.ts` | `expectedDefinitions` + sorted vendor roster |
| `provider-definition-types.test.ts` | `ProviderVendorKey` union if asserted |
| `provider-pipeline-integration.test.ts` | `expectedClassByVendor`, env setup in `beforeEach` |
| `provider-codegen.test.ts` | leaf discovery / generated sync |
| `adapter-resolver.test.ts` | `ADAPTER_MODULES` list if new adapter key |
| `provider-registry.test.ts` | endpoint routing tests using correct `envVar` |

### 3. Shared-server (if discovery or key validation is non-trivial)

`self/apps/shared-server/__tests__/provider-model-discovery.test.ts`:

- Vendor-shaped model list response (if parser edge case)
- `testProviderApiKey` valid/invalid against `healthCheckEndpoint`

### 4. Manual smoke (`pnpm dev:web`, port 4317)

- Provider in API Keys dropdown
- Invalid key fails validation (when auth matters)
- Valid key connects; model picker lists catalog (not only `defaultModelId`)

## Verification Gate

Before PR:

```bash
pnpm --filter @nous/subcortex-providers run generate:providers
pnpm typecheck   # provider package must be green; note pre-existing breaks on integration tip
pnpm lint
pnpm test self/subcortex/providers
pnpm test provider-model-discovery   # if shared-server touched
pnpm build
```

## PR Conventions

- **Title:** `feat(providers): add <Vendor> provider leaf`
- **Scope:** leaf + regenerated catalogs + tests; avoid unrelated fixes
- **Flag pre-existing failures** on integration branch (e.g. `bootstrap.ts` typecheck) — do not scope-creep
- **Do not commit** local process notes (`contribution_readme.md` etc.)
- Conventional commit scoped: `feat(subcortex-providers): ...`

## Decision Tree

```
New vendor API?
├─ OpenAI Chat Completions compatible?
│  ├─ Cloud + API key → groq/ template + fail-closed factory if not OpenAI vendor
│  ├─ Public /v1/models? → add healthCheckEndpoint for key validation (openrouter/)
│  └─ Local server → llama-cpp/ template
├─ Anthropic Messages API? → anthropic/ template (+ implementation.ts)
├─ Ollama? → ollama/ template
└─ CLI subprocess agent? → codex-cli/ template
```

## Maintainer Expectations (from merged OpenRouter PR)

- Leaf shape + tests over drive-by refactors
- Fail closed on shared-protocol credential leaks
- Separate key validation from public model catalogs
- Model discovery compatibility fixes are welcome when minimal and tested
- Parallel provider merges may cause catalog churn — maintainer may resolve at merge
- Broader `ChatCompletionsProvider` cleanup tracked in #413; `nativeToolUse` in #390

## Additional Resources

- [reference.md](reference.md) — copy-paste templates, env var naming, index barrel
- `CONTRIBUTING.md` — Model Provider Adapters section
- `CLAUDE.md` — monorepo commands and provider codegen sharp edges
