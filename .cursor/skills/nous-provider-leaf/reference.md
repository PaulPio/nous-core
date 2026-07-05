# Nous Provider Leaf — Reference

## Groq-shaped cloud leaf (minimal OpenAI-compatible)

Use when the vendor speaks OpenAI Chat Completions, requires an API key, and `/v1/models` validates auth (or you add `healthCheckEndpoint`).

### definition.ts

```ts
import type { ProviderDefinitionLeaf } from '../../schemas/provider-definition.js';

export const ACME_DEFAULT_ENDPOINT = 'https://api.acme.example';
export const ACME_DEFAULT_MODEL_ID = 'acme-default';

export const ACME_PROVIDER_DEFINITION = {
  vendorKey: 'acme',
  displayName: 'Acme',
  providerType: 'text',
  providerClass: 'remote_text',
  protocol: 'chat-completions',
  adapterKey: 'chat-completions',
  defaultEndpoint: ACME_DEFAULT_ENDPOINT,
  defaultModelId: ACME_DEFAULT_MODEL_ID,
  auth: {
    envVar: 'ACME_API_KEY',
    vaultKeyNamespace: 'acme',
    header: { name: 'Authorization', scheme: 'bearer' },
    required: true,
    purpose: 'api_key',
  },
  modelListEndpoint: '/v1/models',
  modelListFormat: 'openai-models',
  capabilities: {
    streaming: true,
    modelListing: true,
    // nativeToolUse intentionally omitted (#390)
  },
  isLocal: false,
} as const satisfies ProviderDefinitionLeaf;

export { ACME_PROVIDER_DEFINITION as providerDefinition };
```

### provider.ts — fail-closed (non-OpenAI vendors)

```ts
import { NousError } from '@nous/shared';
import { ChatCompletionsProvider } from '../../protocols/openai-api/provider.js';
import type { ProviderFactoryModule } from '../../schemas/provider-factory.js';
import { ACME_PROVIDER_DEFINITION } from './definition.js';

const ENV_VAR = ACME_PROVIDER_DEFINITION.auth.envVar!;

export const providerFactory = {
  vendorKey: 'acme',
  create(config, options) {
    const apiKey = options?.apiKey ?? process.env[ENV_VAR];
    if (!apiKey) {
      throw new NousError(
        'Acme API key required — set ACME_API_KEY or pass apiKey option',
        'PROVIDER_AUTH_FAILED',
        { failoverReasonCode: 'PRV-AUTH-FAILURE' },
      );
    }
    return new ChatCompletionsProvider(config, { apiKey });
  },
} as const satisfies ProviderFactoryModule;
```

### adapter.ts

```ts
export {
  chatCompletionsAdapter as providerAdapter,
  createChatCompletionsAdapter,
} from '../../protocols/openai-api/adapter.js';
```

### index.ts

```ts
export { providerAdapter } from './adapter.js';
export {
  ACME_DEFAULT_ENDPOINT,
  ACME_DEFAULT_MODEL_ID,
  ACME_PROVIDER_DEFINITION,
  providerDefinition,
} from './definition.js';
export { providerFactory } from './provider.js';
export { ChatCompletionsProvider } from '../../protocols/openai-api/provider.js';
```

---

## OpenRouter-shaped leaf (public model list)

Add when `/v1/models` returns 200 without valid auth:

```ts
modelListEndpoint: '/v1/models',
modelListFormat: 'openai-models',
// Public catalog — do not use for key validation.
healthCheckEndpoint: '/v1/key',
```

OpenRouter docs: `GET /v1/key` returns 401 for invalid keys, 200 with metadata for valid keys.

---

## Local llama-cpp-shaped leaf

```ts
auth: { required: false, purpose: 'api_key' },
isLocal: true,
providerClass: 'local_text',
healthCheckEndpoint: '/v1/models',  // optional reachability check
```

### provider.ts

```ts
return new ChatCompletionsProvider(config, {
  apiKey: options?.apiKey ?? 'no-auth',
});
```

---

## Env var naming

Pattern: `<VENDOR>_API_KEY` (uppercase, underscores). Examples:

| vendorKey | envVar |
|-----------|--------|
| `groq` | `GROQ_API_KEY` |
| `openrouter` | `OPENROUTER_API_KEY` |
| `anthropic` | `ANTHROPIC_API_KEY` |

`vaultKeyNamespace` matches `vendorKey` (kebab-case). Vault storage key: `api_key_<vaultKeyNamespace>`.

---

## Test template sketch

```ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { NousError } from '@nous/shared';
import { providerFactory, ACME_PROVIDER_DEFINITION } from '../../providers/acme/index.js';
import { deriveBuiltInProviderId } from '../../provider-identity.js';

const MOCK_CONFIG = {
  id: deriveBuiltInProviderId('acme'),
  name: 'Acme',
  type: 'text' as const,
  modelId: ACME_PROVIDER_DEFINITION.defaultModelId,
  isLocal: false,
  capabilities: ['text'],
};

describe('Acme provider leaf', () => {
  const orig = process.env.ACME_API_KEY;
  beforeEach(() => { delete process.env.ACME_API_KEY; });
  afterEach(() => {
    if (orig === undefined) delete process.env.ACME_API_KEY;
    else process.env.ACME_API_KEY = orig;
  });

  it('factory throws without api key', () => {
    expect(() => providerFactory.create(MOCK_CONFIG, {})).toThrow(NousError);
  });

  it('does not use OPENAI_API_KEY', () => {
    process.env.OPENAI_API_KEY = 'wrong';
    expect(() => providerFactory.create(MOCK_CONFIG, {})).toThrow(NousError);
  });
});
```

---

## Aggregate test roster update

In `provider-definitions.test.ts`, add to **both**:

1. `expectedDefinitions` object
2. Sorted array in `contains exactly the current validation roster`

In `provider-pipeline-integration.test.ts`:

```ts
process.env.ACME_API_KEY = 'test-acme-key';
// ...
expectedClassByVendor: { ..., acme: ChatCompletionsProvider }
```

In `provider-registry.test.ts` — if testing endpoint URL resolution, set the **vendor's** env var, not `OPENAI_API_KEY`.

---

## Pitfalls (from production contributions)

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| `undefined` apiKey → `ChatCompletionsProvider` | OpenAI key sent to wrong host | Fail closed in leaf factory |
| Public `/v1/models` used for key test | Invalid keys show as valid in Settings | `healthCheckEndpoint` |
| Strict OpenAI Zod on model list | Only `defaultModelId` in picker | Optional fields in discovery schema + vendor-shaped test |
| Hand-authored UUID | Review rejection | Omit `wellKnownProviderId` |
| Stale `ADAPTER_MODULES` test | CI fail after parallel leaf merge | Update expected adapter list |
| Editing generated catalogs | `check:generated` fails | Run `generate:providers` |
| `nativeToolUse: true` | Premature capability claim | Omit until #390 |
| Process docs in repo | Maintainer cleanup request | Keep local; gitignore `/*.md` covers root md |

---

## Files you should NOT change (standard leaf PR)

- `self/shared/src/interfaces/*` (unless maintainer scopes interface work)
- `protocols/openai-api/provider.ts` (use leaf factory workaround until #413)
- Generated: `provider-definitions.ts`, `provider-factories.ts`, `provider-adapters.ts`
- App/UI code for API-key providers (definition-driven since integration branch)

## Files you MAY change (when justified)

- `self/apps/shared-server/src/provider-model-discovery.ts` — parser compatibility (minimal diff + test)
- Aggregate test rosters — always when adding a vendor

---

## Upstream sync workflow

```bash
git fetch upstream
git checkout -b feat/acme-provider-leaf upstream/feat/contributor-friendly-inference-provider-surface
# implement leaf
pnpm --filter @nous/subcortex-providers run generate:providers
pnpm test self/subcortex/providers
git push -u origin feat/acme-provider-leaf
gh pr create --base feat/contributor-friendly-inference-provider-surface
```

Confirm the current integration branch name with maintainers if `main` has absorbed the provider surface.
