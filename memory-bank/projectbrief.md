# Chartsmith Vercel AI SDK Migration - Project Brief

**Project**: Chartsmith AI SDK Migration  
**Challenge**: Replicated Uncharted Territory Challenge  
**Source**: [github.com/replicatedhq/chartsmith](https://github.com/replicatedhq/chartsmith)

---

## Mission Statement

Migrate Chartsmith from its custom `@anthropic-ai/sdk` implementation to Vercel AI SDK, enabling standardized chat functionality, multi-provider support, and reduced maintenance burden while maintaining all existing features.

---

## What is Chartsmith?

Chartsmith is an open-source AI-powered tool by Replicated that helps developers build better Helm charts for Kubernetes. It provides:

- AI-assisted Helm chart creation and editing
- Natural language interface for chart modifications
- Real-time streaming responses
- Tool calling for file operations and version lookups
- VS Code extension integration

---

## The Challenge

Replace the custom chat UI and LLM implementation with Vercel AI SDK to:

1. **Simplify the frontend**: Replace custom chat components with Vercel AI SDK UI hooks
2. **Modernize the backend**: Replace direct Anthropic SDK calls with AI SDK Core's unified API
3. **Enable multi-provider support**: Access multiple LLM providers through OpenRouter
4. **Reduce maintenance burden**: Use standardized, well-maintained toolkit

---

## Core Requirements (From Replicated_Chartsmith.md)

### Must Have
1. Replace custom chat UI with Vercel AI SDK
2. Migrate from direct `@anthropic-ai/sdk` usage to AI SDK Core
3. Maintain all existing chat functionality (streaming, messages, history)
4. Keep existing system prompts and behavior (user roles, chart context)
5. All existing features continue to work (tool calling, file context, etc.)
6. Tests pass (or are updated to reflect new implementation)

### Nice to Have
1. Demonstrate easy provider switching (show how to swap Anthropic for OpenAI)
2. Improve the streaming experience using AI SDK optimizations
3. Simplify state management by leveraging AI SDK's built-in patterns

---

## Submission Requirements

1. **Pull Request** into the `replicatedhq/chartsmith` repo
2. **Documentation**: Update `ARCHITECTURE.md` or `chartsmith-app/ARCHITECTURE.md`
3. **Tests**: Ensure existing tests pass or update them
4. **Demo Video**: Show application starting, chart creation, streaming, and code walkthrough

---

## PR Structure (3 PRs)

| PR | Name | Focus | Deliverables |
|----|------|-------|--------------|
| **PR1** | AI SDK Foundation | Days 1-3 | New `/api/chat` route, `useChat` hook, provider selection UI, streaming |
| **PR1.5** | Migration & Feature Parity | Days 3-4 | Go HTTP server, 6 AI SDK tools, system prompts, remove @anthropic-ai/sdk |
| **PR2** | Validation Agent | Days 4-6 | Chart validation pipeline, `validateChart` tool, live provider switching |

---

## Key Architectural Decision

**Go does NOT make LLM calls after PR1.5.**

- **Node/AI SDK** = All "thinking" (LLM calls, tool orchestration, reasoning)
- **Go** = Pure application logic (chart CRUD, file operations, validation)

This creates a clean separation: intelligence layer (Node) vs. execution layer (Go).

---

## Success Metrics

1. Chat functionality equivalent to current implementation
2. Streaming works with no visible degradation
3. Provider selector allows choosing between models
4. All existing tools work (file context, chart operations)
5. "Create a new chart via chat" works end-to-end
6. VS Code extension authentication works
7. No console errors during normal operation

---

## References

- Vercel AI SDK: [ai-sdk.dev/docs](https://ai-sdk.dev/docs)
- OpenRouter: Meta-provider for multiple LLM APIs
- Chartsmith GitHub: [github.com/replicatedhq/chartsmith](https://github.com/replicatedhq/chartsmith)

---

*This brief is the foundation document for the Chartsmith AI SDK migration.*

