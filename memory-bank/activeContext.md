# Chartsmith Active Context

**Purpose**: Track current work focus, recent changes, next steps, and active decisions.

---

## Current Phase

**Status**: Pre-Implementation (Planning Complete)

All PRDs have been created and validated. Ready to begin implementation of PR1.

---

## PR Structure & Timeline

| PR | Name | Timeline | Status |
|----|------|----------|--------|
| PR1 | AI SDK Foundation | Days 1-3 | **Ready for Implementation** |
| PR1.5 | Migration & Feature Parity | Days 3-4 | Documented, awaiting PR1 |
| PR2 | Validation Agent | Days 4-6 | Documented, awaiting PR1.5 |

---

## Current Work Focus

### Immediate Priority: PR1 Implementation

**Day 1 Tasks (Foundation)**:
1. Create feature branch from main
2. Install new dependencies (`ai`, `@ai-sdk/react`, `@openrouter/ai-sdk-provider`)
3. Add environment variables to `.env.example`
4. Create provider factory module (`lib/ai/provider.ts`)
5. Create model configuration (`lib/ai/models.ts`)
6. Create API route with streamText (`app/api/chat/route.ts`)
7. Test basic streaming response
8. Verify OpenRouter connectivity

**Day 2 Tasks (Component Migration)**:
1. Create ProviderSelector component
2. Migrate Chat component to useChat hook
3. Update message rendering for parts-based format
4. Test provider switching in new conversation

**Day 3 Tasks (Polish & Testing)**:
1. Update/fix failing tests
2. Add new unit tests
3. Run full test suite
4. Update ARCHITECTURE.md
5. Code cleanup and review prep
6. Final testing and bug fixes

---

## Key Technical Decisions (Resolved)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Migration approach | Hybrid: New system primary, legacy deprecated | Meets "replace" requirement with safety net |
| Go LLM responsibility | None after PR1.5 | Clean separation: Node thinks, Go executes |
| Tool migration | 4 tools (3 existing + 1 new) | Feature parity; workspace creation handled by homepage flow |
| Go communication | HTTP endpoints (not PostgreSQL queue) | Synchronous tool execution |
| PR structure | 3 PRs (Foundation → Parity → Validation) | Clear deliverables, risk isolation |
| System prompts | Migrate to Node | Colocation with LLM calls |

**Note on Tools (Updated Dec 3)**: After codebase analysis, determined that workspace/chart creation happens via homepage TypeScript flow BEFORE chat begins. AI SDK needs only 4 tools: getChartContext (TypeScript-only), textEditor, latestSubchartVersion, latestKubernetesVersion.

---

## Environment Setup Required

### New Environment Variables

```env
# Required: OpenRouter API Key
OPENROUTER_API_KEY=sk-or-v1-xxxxx

# Optional: Direct OpenAI fallback
OPENAI_API_KEY=sk-xxxxx

# Configuration
DEFAULT_AI_PROVIDER=openai
DEFAULT_AI_MODEL=openai/gpt-4o

# PR1.5: Go backend URL
GO_BACKEND_URL=http://localhost:8080
```

---

## Files to Create (PR1)

### New Files

| File | Purpose |
|------|---------|
| `lib/ai/provider.ts` | Provider factory for OpenRouter |
| `lib/ai/models.ts` | Model definitions and configuration |
| `lib/ai/config.ts` | AI configuration |
| `app/api/chat/route.ts` | New chat API route |
| `components/chat/ProviderSelector.tsx` | Model selector UI |

### Files to Modify

| File | Changes |
|------|---------|
| `package.json` | Add new dependencies, remove old |
| `components/Chat.tsx` | Replace with useChat hook |
| `components/Message.tsx` | Update for parts-based rendering |
| `chartsmith-app/ARCHITECTURE.md` | Document new AI SDK integration |

### Files to Delete

| File | Reason |
|------|--------|
| `lib/llm/prompt-type.ts` | Orphaned code - never called |

---

## Integration Points

### What PR1 Creates (NEW)

- `/api/chat/route.ts` - New Next.js API route using `streamText`
- `lib/ai/provider.ts` - Provider factory for OpenRouter
- `components/chat/ProviderSelector.tsx` - Model selection UI
- Chat components using `useChat` hook from `@ai-sdk/react`

### What PR1 Preserves (EXISTING)

- Go backend and its LLM calls (unchanged until PR1.5)
- Existing workspace APIs and Jotai state management
- Centrifugo real-time updates for existing features
- Existing tool implementations in Go

---

## Active Considerations

### Streaming Coexistence

PR1 introduces a second streaming system:

| System | Protocol | Used By |
|--------|----------|---------|
| **Existing** | Centrifugo WebSocket | Workspace operations, plan progress |
| **NEW (PR1)** | AI SDK Data Stream (SSE) | New `/api/chat` responses |

Both systems run in parallel without conflict.

### Provider Selection Behavior

**PR1 Behavior**:
- Selector visible when message history is empty
- Hidden after first message sent
- Selection locked for duration of conversation

**PR2 Enhancement**:
- Selector always visible (Cursor-style)
- Can switch mid-conversation
- History preserved on switch

---

## Blocking Dependencies

### PR1 Dependencies
- OpenRouter API key
- Access to chartsmith repository

### PR1.5 Dependencies
- PR1 merged
- Go backend accessible
- PostgreSQL database running

### PR2 Dependencies
- PR1.5 merged
- helm CLI (v3.x) installed
- kube-score CLI (v1.16+) installed

---

## Test Strategy

### PR1 Test Focus
- Provider factory returns correct models
- Message formatting functions work correctly
- useChat hook integration with API route
- Streaming end-to-end with mock provider
- VS Code extension auth works

### Success Criteria (PR1)
- [ ] Chat functionality equivalent to current implementation
- [ ] Streaming works with no visible degradation
- [ ] Provider selector allows choosing between GPT-4o and Claude
- [ ] All existing tests pass (or updated appropriately)
- [ ] No console errors during normal operation

---

## Next Actions (Start Here)

1. Clone chartsmith repository
2. Create feature branch `feat/vercel-ai-sdk-migration`
3. Install dependencies:
   ```bash
   npm install ai @ai-sdk/react @openrouter/ai-sdk-provider
   ```
4. Set up environment variables
5. Create `lib/ai/provider.ts`
6. Create `app/api/chat/route.ts`
7. Test basic streaming response

---

*This document is the starting point for each work session. Update after completing tasks.*

