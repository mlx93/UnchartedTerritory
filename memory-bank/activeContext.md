# Chartsmith Active Context

**Purpose**: Track current work focus, recent changes, next steps, and active decisions.

---

## Current Phase

**Status**: PR2.0 ✅ Complete; PR3.0 ✅ Complete; PR3.1 ✅ Complete; PR3.2 ✅ Complete; PR3.3 ✅ Complete

PR2.0 reintegrated AI SDK into the main workspace path (`/workspace/[id]`) behind a feature flag:
- Adapter pattern bridges AI SDK to existing Message format
- Feature flag `NEXT_PUBLIC_USE_AI_SDK_CHAT` for rollout/rollback
- Persona propagation (auto/developer/operator) affects system prompts
- Cancel support with partial response persistence
- Centrifugo coordination prevents streaming conflicts

PR3.0 (Complete): Full feature parity implementation following Option A strategy.
- Plan workflow: buffer `textEditor` create/str_replace tool calls → Go `/api/plan/create-from-tools` → sets `response_plan_id` → existing `PlanChatMessage` renders with Proceed/Ignore.
- Proceed executes buffered tool calls; Ignore marks plan ignored.
- Intent routing: AI SDK with Go intent classification gates off-topic/proceed/render.
- Buffered tool storage: persisted to `workspace_plan.buffered_tool_calls` JSONB column.
- Page reload guard: warns users before leaving with unsaved changes or active streaming.
- Rule-based followups: auto-generates "Render the chart" actions after file operations.
- Rollback support: first message of revision includes rollback link after commit.

PR3.1 (Complete): AI SDK UX improvements and buffered tool calls fix.
- Centrifugo metadata updates for streaming messages.
- Status validation and reset-on-failure in proceedPlanAction.
- Accept/Reject buttons visibility fixes in AI SDK mode.

PR3.2 (Complete): Two-phase plan/execute workflow parity.
- Centered chat layout until plan execution.
- Two-phase workflow: plan generation (no tools) → execution phase (with tools).
- Intent classification endpoint integration.

PR3.3 (Complete): AI SDK execution path for text-only plans.
- `execute-via-ai-sdk.ts` server action for executing plans without buffered tool calls.
- LLM-based file extraction from plan descriptions.
- Dynamic action file list building during execution (mirrors Go behavior).
- Execution prompt refinements for better tool call generation.

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
| PR3.0 | Feature Parity Implementation | Day 8 | ✅ **Complete** |
| PR3.1 | AI SDK UX Improvements | Day 8 | ✅ **Complete** |
| PR3.2 | Two-Phase Plan/Execute Workflow | Day 8 | ✅ **Complete** |
| PR3.3 | AI SDK Execution Path | Day 8 | ✅ **Complete** |
| PR2 | Validation Agent | Days 9-11 | **Ready for Implementation** |

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

### PR3.0-3.3: Feature Parity & Execution Path ✅

#### PR3.0: Full Feature Parity
- Intent classification client (`lib/ai/intent.ts`)
- Plan creation client (`lib/ai/plan.ts`)
- Conversion client (`lib/ai/conversion.ts`)
- Tool call buffering (`lib/ai/tools/toolInterceptor.ts`, `bufferedTools.ts`)
- K8s conversion tool (`lib/ai/tools/convertK8s.ts`)
- Proceed/ignore actions (`lib/workspace/actions/proceed-plan.ts`)
- Go endpoints: `/api/intent/classify`, `/api/plan/create-from-tools`, `/api/conversion/convert-k8s`
- Page reload guard in WorkspaceContent
- Rule-based followup action generation
- Rollback field population on commit

#### PR3.1: UX Improvements
- Centrifugo metadata update fixes
- Status validation in proceedPlanAction
- Accept/Reject button visibility fixes

#### PR3.2: Two-Phase Workflow
- Centered chat layout until plan execution
- Intent classification integration
- Two-phase plan/execute separation

#### PR3.3: Text-Only Plan Execution
- `execute-via-ai-sdk.ts` server action
- LLM-based file extraction from plan descriptions
- Dynamic action file list building
- Execution prompt refinements

### Previous PRs (Summary)

#### PR1.7: Deeper System Integration ✅
- **Centrifugo Integration**: Real-time file updates via `artifact-updated` events published from Go textEditor handler
- **Pending Changes UI**: Yellow dots in file explorer, pending count in tab bar, diff stats showing additions/deletions
- **Revision Tracking**: Commit/Discard functionality via `commit-pending-changes.ts` and `discard-pending-changes.ts`
- **Prerequisite Bug Fixes**: 
  - Fixed double processing (ChatMessageIntent.NON_PLAN)
  - Fixed inconsistent content handling (`AddFileToChartPending()` writes to `content_pending`)
- **Files Created**: `lib/workspace/actions/commit-pending-changes.ts`, `lib/workspace/actions/discard-pending-changes.ts`
- **Files Modified**: `pkg/workspace/chart.go` (AddFileToChartPending), `pkg/api/handlers/editor.go` (Centrifugo events), `pkg/workspace/file.go` (GetFileIDByPath), `app/test-ai-chat/[workspaceId]/client.tsx` (useCentrifugo, pending UI)

#### PR1.6/1.61/1.65: Feature Parity ✅
- Simplified system prompt
- Landing page with workspace creation
- Three-panel layout
- Body parameter hotfix

---

## Key Technical Decisions

### PR2.0 Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Integration approach | Adapter pattern | 75% less effort than rebuilding 27+ features |
| Rollout strategy | Feature flag | Instant rollback, safe gradual rollout |
| Review mechanism | Commit/Discard | Already working, avoids Go changes |
| Interval tracking | useRef | React best practice for mutable values |
| Centrifugo coordination | Skip streaming updates | Prevents state conflicts between AI SDK and Centrifugo |

### PR1.7 Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Content handling | AddFileToChartPending() | Separate function preserves K8s conversion workflow (needs direct content writes) |
| Real-time updates | Centrifugo events | Reuses existing infrastructure, no refetch needed |
| Revision creation | Commit action | Reuses existing accept/reject patch functions |

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

### PR3.0-3.3 Post-Implementation Tasks:
1. Monitor plan execution success rates (buffered vs text-only paths)
2. Test rollback functionality end-to-end
3. Verify followup action generation accuracy
4. Monitor intent classification accuracy
5. Consider deprecating Go LLM execution path once AI SDK path is stable

---

## Research Documents

| Document | Purpose |
|----------|---------|
| `docs/PR3.0_COMPLETION_REPORT_FINAL.md` | Full PR3.0 implementation details |
| `docs/PR3.0_AI_SDK_PLAN_INTEGRATION_FLOW.md` | PR3.0 plan integration flow |
| `docs/PR3.1_AI_SDK_UX_IMPROVEMENTS.md` | PR3.1 UX improvements |
| `docs/PR2.0_COMPLETION_REPORT.md` | Full PR2.0 implementation details |
| `PRDs/PR2.0_REINTEGRATION_Tech_PRD.md` | PR2.0 technical specification |
| `PRDs/PR2.0_REINTEGRATION_IMPLEMENTATION_PLAN.md` | PR2.0 implementation plan |
| `PRDs/PR3.0_FEATURE_PARITY_IMPLEMENTATION_PLAN.md` | PR3.0 implementation plan |

---

*This document is the starting point for each work session. Last updated: Dec 6, 2025 (PR3.0-3.3 complete)*
