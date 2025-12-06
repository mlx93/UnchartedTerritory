# Chartsmith Active Context

**Purpose**: Track current work focus, recent changes, next steps, and active decisions.

---

## Current Phase

**Status**: PR2.0 ✅ Complete; PR3.0 Parity Plan ✅ Ready to implement

PR2.0 reintegrated AI SDK into the main workspace path (`/workspace/[id]`) behind a feature flag:
- Adapter pattern bridges AI SDK to existing Message format
- Feature flag `NEXT_PUBLIC_USE_AI_SDK_CHAT` for rollout/rollback
- Persona propagation (auto/developer/operator) affects system prompts
- Cancel support with partial response persistence
- Centrifugo coordination prevents streaming conflicts

PR3.0 strategy (Option A parity): Accept different AI prose while matching plan/flow parity.
- Plan workflow: buffer `textEditor` create/str_replace tool calls → Go `/api/plan/create-from-tools` → sets `response_plan_id` → existing `PlanChatMessage` renders with Proceed/Ignore.
- Proceed executes buffered tool calls; Ignore marks plan ignored.
- Intent routing: still AI SDK; Go intent classification only gates off-topic/proceed/render.
- Buffered tool storage: in-memory during stream; persisted to `workspace_plan.buffered_tool_calls` at plan creation (JSONB).
- DB migration: add `workspace_plan.buffered_tool_calls` JSONB + index; rollback drops column.
- Parity decision: Option A (same UI/flow, prose may differ).

---

## PR Structure & Timeline

| PR | Name | Timeline | Status |
|----|------|----------|--------|
| PR1 | AI SDK Foundation | Days 1-3 | ✅ **Complete** |
| PR1.5 | Tool Integration & Go HTTP | Days 3-4 | ✅ **Complete** |
| PR1.6 | Feature Parity | Day 5 | ✅ **Complete** |
| PR1.61 | Body Parameter Hotfix | Day 5 | ✅ **Complete** |
| PR1.65 | UI Feature Parity | Day 5 | ✅ **Complete** |
| PR1.7 Prereq | Bug Fixes for PR1.7 | Day 5 | ✅ **Complete** |
| PR1.7 | Deeper System Integration | Day 6 | ✅ **Complete** |
| PR2.0 | AI SDK Reintegration | Day 7 | ✅ **Complete** |
| PR2 | Validation Agent | Days 8-10 | **Ready for Implementation** |

---

## Completed Work Summary

### PR2.0: AI SDK Reintegration ✅

PR2.0 integrates AI SDK into the main workspace path, replacing Go worker chat with AI SDK streaming while preserving all existing UI.

#### New Files Created

| File | Purpose |
|------|---------|
| `lib/chat/messageMapper.ts` | UIMessage ↔ Message format conversion |
| `lib/chat/__tests__/messageMapper.test.ts` | 33 unit tests (all passing) |
| `hooks/useAISDKChatAdapter.ts` | Adapter bridging AI SDK to existing patterns |
| `hooks/useLegacyChat.ts` | Wrapper for legacy Go worker path |

#### Modified Files

| File | Changes |
|------|---------|
| `app/api/chat/route.ts` | Added persona parameter, prompt selection |
| `lib/ai/prompts.ts` | Added developer/operator prompts |
| `components/ChatContainer.tsx` | Feature flag, adapter, streaming indicators |
| `hooks/useCentrifugo.ts` | Skip updates for streaming messages |
| `ARCHITECTURE.md` | PR2.0 documentation section |

#### Key Features

1. **Feature Flag**: `NEXT_PUBLIC_USE_AI_SDK_CHAT=true` enables AI SDK path
2. **Adapter Pattern**: Converts UIMessage to Message format seamlessly
3. **Status Mapping**: submitted→thinking, streaming→streaming, ready→complete
4. **Persona Support**: auto/developer/operator affect system prompts
5. **Cancel State**: Partial responses persisted on cancel
6. **Centrifugo Coordination**: `currentStreamingMessageIdAtom` prevents conflicts

#### Bug Fixes (Pre-existing)

| Fix | File |
|-----|------|
| React hooks dependency warnings | `components/TtlshModal.tsx` |

### Previous PRs (Summary)

#### PR1.7: Deeper System Integration ✅
- Real-time file updates via Centrifugo
- Pending changes UI indicators (yellow dots)
- Commit/Discard functionality

#### PR1.6/1.61/1.65: Feature Parity ✅
- Simplified system prompt
- Landing page with workspace creation
- Three-panel layout
- Body parameter hotfix

---

## Key Technical Decisions (PR2.0)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Integration approach | Adapter pattern | 75% less effort than rebuilding 27+ features |
| Rollout strategy | Feature flag | Instant rollback, safe gradual rollout |
| Review mechanism | Commit/Discard | Already working, avoids Go changes |
| Interval tracking | useRef | React best practice for mutable values |

---

## Environment Setup

### Current Environment Variables

```env
# Direct API Keys (preferred)
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-xxx

# Fallback
OPENROUTER_API_KEY=sk-or-v1-xxxxx

# Configuration (defaults)
DEFAULT_AI_PROVIDER=anthropic
DEFAULT_AI_MODEL=anthropic/claude-sonnet-4-20250514

# Go backend URL
GO_BACKEND_URL=http://localhost:8080

# Centrifugo
NEXT_PUBLIC_CENTRIFUGO_ADDRESS=ws://localhost:8000/connection/websocket
CENTRIFUGO_TOKEN_HMAC_SECRET=change.me

# PR2.0: AI SDK Chat Feature Flag
NEXT_PUBLIC_USE_AI_SDK_CHAT=false  # Set to 'true' to enable
```

---

## Next Actions

### NEXT: PR2 Implementation (Validation Agent)
1. Read `PRDs/PR2_Product_PRD.md` and `PRDs/PR2_Tech_PRD.md`
2. Verify helm and kube-score CLI tools are installed
3. Create `pkg/validation/` package structure
4. Create validation endpoint in Go
5. Create validateChart tool in TypeScript

### PR2.0 Post-Merge Tasks:
1. Test in staging with `NEXT_PUBLIC_USE_AI_SDK_CHAT=true`
2. Monitor error rates and response times
3. After stable period: Remove `/test-ai-chat` directory

---

## Research Documents

| Document | Purpose |
|----------|---------|
| `docs/PR2.0_COMPLETION_REPORT.md` | Full PR2.0 implementation details |
| `PRDs/PR2.0_REINTEGRATION_Tech_PRD.md` | PR2.0 technical specification |
| `PRDs/PR2.0_REINTEGRATION_IMPLEMENTATION_PLAN.md` | PR2.0 implementation plan |

---

*This document is the starting point for each work session. Last updated: Dec 5, 2025 (PR2.0 complete)*
