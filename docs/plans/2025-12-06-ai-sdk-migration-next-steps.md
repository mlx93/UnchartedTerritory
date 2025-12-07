# AI SDK Migration: Current State & Next Steps

**Date**: 2025-12-06
**Status**: Partial Implementation - Feature Parity Achieved, Full Migration Pending
**Branch**: `myles/vercel-ai-sdk-migration`

---

## Executive Summary

The two-phase plan/execute workflow now has **feature parity** with the main branch, but uses a **hybrid architecture**:
- **Frontend (Plan Phase)**: Uses Vercel AI SDK ✅
- **Backend (Execute Phase)**: Still uses Go + Anthropic SDK directly ❌

This document outlines the current issues, requirements gap analysis, and strategies for completing the migration.

---

## Current Issues

### Issue 1: Duplicate Text in Plan UI

**Problem**: The streaming plan text appears twice in the UI:
1. First as the streaming AI response in the chat
2. Again in the "Proposed Plan" card below

**Root Cause**: The plan description is stored in `workspace_plan.description` and then displayed via `PlanChatMessage` component, but the same text is also rendered as the chat message response.

**Expected Behavior (Main Branch)**:
- Chat shows the plan description text (what the AI will do)
- "Proposed Plan" card shows a **list of files** to be created/modified (not the text again)

**Why This Happens**:
- In main branch: `CreateExecutePlan` generates `<chartsmithActionPlan>` XML tags listing files
- In myles branch: We store the plan text as description, and the file list is only generated AFTER clicking Proceed

**Potential Solutions**:
1. **Parse plan text for file references** - Extract file names from the plan text and display them in the Proposed Plan card
2. **Separate plan text from file list** - Don't show plan text in the Proposed Plan card, only show it when file list is available
3. **Generate file list during plan phase** - Make an additional AI call to extract the file list from the plan text
4. **Hide duplicate content** - If `responsePlanId` exists and plan has no action files, don't show the Proposed Plan card

### Issue 2: Backend Still Uses Anthropic SDK Directly

**Current Architecture**:
```
User Message → AI SDK (TypeScript) → Plan Text Response
     ↓
User Clicks Proceed
     ↓
createRevisionAction → Go Worker → Anthropic SDK → File Creation
```

**This violates the requirement**: "Migrate from direct @anthropic-ai/sdk usage to AI SDK Core"

---

## Requirements Gap Analysis

| Requirement | Status | Notes |
|------------|--------|-------|
| **Must Have 1**: Replace custom chat UI with Vercel AI SDK | ✅ Complete | Using `useChat` from `@ai-sdk/react` |
| **Must Have 2**: Migrate from @anthropic-ai/sdk to AI SDK Core | ⚠️ **PARTIAL** | Frontend uses AI SDK, backend still uses Go + Anthropic SDK |
| **Must Have 3**: Maintain streaming, messages, history | ✅ Complete | Working correctly |
| **Must Have 4**: Keep existing system prompts and behavior | ✅ Complete | Prompts preserved in `prompts.ts` |
| **Must Have 5**: Tool calling, file context work | ✅ Complete | `textEditor` tool works |
| **Must Have 6**: Tests pass | ⚠️ Need verification | |
| **Nice to Have 1**: Easy provider switching | ✅ Supported | `getModel()` supports multiple providers |
| **Nice to Have 2**: Improved streaming | ✅ Complete | AI SDK optimizations |
| **Nice to Have 3**: Simplified state management | ✅ Complete | Using AI SDK patterns |

### Key Gap: Backend LLM Calls

The Go backend's `pkg/llm/` package still makes direct calls to Anthropic:
- `pkg/llm/execute-action.go` - Uses `anthropic.Messages.NewStreaming()` for `text_editor` tool
- `pkg/llm/execute-plan.go` - Uses `anthropic.Messages.NewStreaming()` for detailed plan generation
- `pkg/llm/initial-plan.go` - Uses Anthropic for initial plan generation
- `pkg/llm/plan.go` - Uses Anthropic for update plans

---

## Strategies for Full Migration

### Strategy 1: Execute via AI SDK (Recommended)

**Approach**: When user clicks "Proceed", instead of triggering Go worker, call AI SDK with tools enabled.

**Implementation**:
```typescript
// In PlanChatMessage.tsx handleProceed()
if (plan.bufferedToolCalls?.length === 0) {
  // Text-only plan: Call AI SDK with execution prompt + tools
  await executeWithAISDK(session, workspaceId, plan.description);
} else {
  // Buffered tool calls: Execute directly
  await proceedPlanAction(session, plan.id, wsId, revisionNumber);
}
```

**New function `executeWithAISDK()`**:
1. Send the plan description + "Now execute this plan" instruction
2. Enable `textEditor` tool
3. Stream tool calls to UI
4. Execute tools via existing `callGoEndpoint('/api/tools/editor', ...)`

**Pros**:
- Fully migrates to AI SDK
- Leverages existing tool infrastructure
- Maintains streaming UX

**Cons**:
- Requires new execution flow
- May differ from main branch behavior
- Need to handle multi-step tool calls

### Strategy 2: Parallel Path Development

**Approach**: Keep both paths working, gradually migrate features to AI SDK path.

**Current State**:
- `chatMessageId` provided → Uses AI SDK buffered tools path
- `chatMessageId` not provided → Uses legacy Go worker path

**Next Steps**:
1. Add `useAISDKExecution` flag to plan records
2. When flag is true, execution uses AI SDK streamText with tools
3. Gradually test and enable for all plans

**Pros**:
- Low risk - can fall back to legacy
- Incremental migration
- Easy A/B testing

**Cons**:
- Maintaining two codepaths
- Complexity

### Strategy 3: Go Backend as Tool Executor Only

**Approach**: Go backend only handles tool execution (file read/write), all LLM calls go through AI SDK.

**Implementation**:
1. Remove LLM calls from Go backend
2. AI SDK handles all prompt/response streaming
3. Go backend exposes REST endpoints for file operations
4. AI SDK calls Go endpoints when tools are invoked

**Current State** (partially implemented):
- `/api/tools/editor` - Go endpoint for file operations ✅
- AI SDK `textEditor` tool calls this endpoint ✅

**What's Missing**:
- The "Proceed" flow still calls Go worker which calls Anthropic directly
- Need to route execution through AI SDK instead

**Pros**:
- Clean separation of concerns
- All AI logic in TypeScript
- Easy to swap providers

**Cons**:
- Significant refactor of proceed flow
- Need to replicate Go's execution logic in TypeScript

---

## Recommended Approach

### Phase A: Fix Duplicate Text Issue (Quick Win)

1. Modify `PlanChatMessage` to not show plan description if it matches the chat message response
2. Or: Only show "Proposed Plan" card when `actionFiles.length > 0`

### Phase B: AI SDK Execution Path

1. Create new `executeViaAISDK()` function in TypeScript
2. When plan has empty `bufferedToolCalls`, use this path instead of Go worker
3. Use existing `textEditor` tool with `createTools()`
4. Stream results to UI using existing patterns

### Phase C: Deprecate Go LLM Calls

1. Once AI SDK execution is stable, mark Go LLM functions as deprecated
2. Remove Go LLM dependencies gradually
3. Keep Go backend for file operations only

---

## Technical Notes

### Current File Flow (Hybrid)

```
1. User: "create nginx deployment"
2. AI SDK: streamText() with PLAN prompt (no tools)
3. AI Response: Plan description text
4. Plan created: workspace_plan with description
5. User clicks: "Proceed"
6. Go Worker: CreateExecutePlan() - generates <chartsmithActionPlan> tags
7. Go Worker: apply_plan() - calls Anthropic for each file
8. Files stream to UI via Centrifugo
```

### Desired File Flow (Full AI SDK)

```
1. User: "create nginx deployment"
2. AI SDK: streamText() with PLAN prompt (no tools)
3. AI Response: Plan description text
4. Plan created: workspace_plan with description
5. User clicks: "Proceed"
6. AI SDK: streamText() with EXECUTION prompt + textEditor tool
7. AI generates tool calls for each file
8. Tool calls executed via /api/tools/editor
9. Files stream to UI via AI SDK onToolResult
```

### Key Components

| Component | Current | Target |
|-----------|---------|--------|
| Plan Generation | AI SDK ✅ | AI SDK ✅ |
| Plan Storage | Go Backend | Go Backend (keep) |
| File List Generation | Go + Anthropic | AI SDK + parsing |
| File Execution | Go + Anthropic | AI SDK + Go tool endpoints |
| File Streaming | Centrifugo | AI SDK stream + Centrifugo |

---

## Files to Modify

### For Issue 1 (Duplicate Text):
- `chartsmith-app/components/PlanChatMessage.tsx` - Conditional rendering
- `chartsmith-app/components/ChatMessage.tsx` - Hide duplicate content

### For Strategy 1 (AI SDK Execution):
- `chartsmith-app/components/PlanChatMessage.tsx` - New handleProceed logic
- `chartsmith-app/lib/ai/prompts.ts` - Add EXECUTION prompt
- `chartsmith-app/lib/workspace/actions/execute-via-ai-sdk.ts` - New action
- `chartsmith-app/app/api/chat/route.ts` - Handle execution intent

### For Strategy 3 (Go as Tool Executor):
- No changes to Go backend (already has `/api/tools/editor`)
- Focus on TypeScript AI SDK implementation

---

## Open Questions

1. **Should we maintain backward compatibility with Go worker?**
   - Pro: Fallback option, less risky
   - Con: Two codepaths to maintain

2. **How to handle multi-file operations?**
   - AI SDK's `stopWhen: stepCountIs(N)` limits tool calls
   - May need higher limit or loop for large charts

3. **Should file list be generated in plan phase or execution phase?**
   - Main branch: Execution phase (after Proceed)
   - Could move to plan phase for better UX

4. **Provider switching for execution?**
   - Plan phase already supports multiple providers
   - Execution needs same flexibility

---

## References

- Implementation Plan: `/docs/plans/2025-12-06-two-phase-plan-execute-parity.md`
- Parity Research: `/docs/testing/MYLES_BRANCH_PARITY_RESEARCH.md`
- Parity Analysis: `/docs/testing/MYLES_BRANCH_PARITY_ANALYSIS.md`
- Main Branch Test: `/docs/testing/MAIN_BRANCH_NGINX_DEPLOYMENT_TEST.md`
- Requirements: `/Replicated_Chartsmith.md`

---

## Next Session Checklist

- [ ] Fix duplicate text issue in PlanChatMessage
- [ ] Implement AI SDK execution path for text-only plans
- [ ] Test provider switching for execution phase
- [ ] Verify all features work end-to-end
- [ ] Run test suite and fix failures
- [ ] Document architectural decision to keep/remove Go LLM calls
