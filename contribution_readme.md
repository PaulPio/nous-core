# Contribution [1]: Adapter: OpenRouter Model Provider #306


**Contribution Number:** 1
**Student:** Paul Piotrowski
**Issue:** https://github.com/orthogonalhq/nous-core/issues/306
**Status:** [Phase I Completed / Phase II Completed / Phase III In Progress / Phase IV] [In Progress / Complete]

---

## Why I Chose This Issue

Its interest me because it looks like a proper challenge for a beginer and its related with AI and AI Agents

---

## Understanding the Issue

### Problem Description

I have to add the Provider for the AI agent for open router to nouse core

### Expected Behavior

Be able to use an API key for Open router

### Current Behavior

There is no support for it

### Affected Components

The src folder that includes the setup for the different LLM models

---

## Reproduction Process

### Environment Setup
setup the environement using node
the setup was made installing the dependences using pnpm install and build, no issues reported so far
### Steps to Reproduce

1. Download the npm modules
2. Build the program
3. Navigate across the site at http://localhost:4317/ to observe if the site its working properly
4. **Expected:** Form to add Openrouter API Key
5. **Actual:** No integration at all

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** 
![1781743413121](image/contribution_readme(1)/1781743413121.png)
- **My findings:** There is no openrouter provider function for llm

---

## Solution Approach

### Analysis

The Openrouter integration for LLM provider has not been implemented yet

### Proposed Solution

Will navegated across the codebase

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Nous currently lacks a selectable OpenRouter provider. OpenRouter is an OpenAI-compatible aggregator; it can be implemented by reusing the existing ChatCompletionsProvider without modifying core interfaces.

**Match:** 
    Use existing infrastructure in self/subcortex/providers/src/:
    
    Protocol: Reuse ChatCompletionsProvider (from src/protocols/openai-api/).
    
    Pattern: Mirror the existing leaf structure used by providers/openai/.
    
    Registry: Utilize the existing generate:providers codegen script to register the new vendor.

**Plan:** [Step-by-step implementation plan]
1. Create src/providers/openrouter/ containing four files:
    definition.ts: Define OPENROUTER_PROVIDER_DEFINITION (unique UUID ...004, key openrouter).

    adapter.ts: Re-export chatCompletionsAdapter.

    provider.ts: Factory using ChatCompletionsProvider.

    index.ts: Barrel file exporting the above.

2. Generate: Run pnpm --filter @nous/subcortex-providers run generate:providers to update aggregates.

3. Public Export: Update src/index.ts to surface OPENROUTER_PROVIDER_DEFINITION.

4. Test: Create src/__tests__/openrouter-provider.test.ts to verify endpoint construction, streaming/parsing, error mapping (401/429), and registry integration.

**Implement:** Branch (Feature/OpenRouterModelProvider)
Task: Follow the directory structure: self/subcortex/providers/src/providers/openrouter/

**Review:** 
[ ] Does definition.ts use a unique wellKnownProviderId?

[ ] Did generate:providers correctly add the vendor to src/provider-factories.ts?

[ ] Are interfaces untouched? (Strict adherence to scope boundary).

**Evaluate:** [How will you verify it works?]
pnpm run check:generated (Validate sync).

pnpm test (Verify openrouter-provider.test.ts and existing providers).

pnpm build (Confirm clean compilation).

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
