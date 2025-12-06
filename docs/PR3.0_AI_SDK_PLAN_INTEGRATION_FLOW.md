# PR3.0: AI SDK Plan Integration Flow

## Overview

This document describes how to integrate the AI SDK chat path with the existing Go plan system to achieve feature parity with the main branch.

## Architecture Flow

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   AI SDK Chat   │────▶│  Tool Interceptor │────▶│  Go /api/plan/  │
│  (textEditor)   │     │  (buffer calls)   │     │ create-from-tools│
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                          │
                        ┌──────────────────┐              │
                        │   Centrifugo     │◀─────────────┘
                        │  (plan update)   │
                        └────────┬─────────┘
                                 │
                        ┌────────▼─────────┐
                        │ PlanChatMessage  │  ← Existing component!
                        │    (renders)     │
                        └────────┬─────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
       ┌──────▼──────┐    ┌──────▼──────┐   ┌──────▼──────┐
       │   Proceed   │    │   Ignore    │   │  Follow-up  │
       │ (execute)   │    │ (mark skip) │   │   (chat)    │
       └─────────────┘    └─────────────┘   └─────────────┘
```

## Detailed Flow

### Step 1: AI SDK Receives Message
- User sends: "create a simple nginx deployment"
- `/api/chat` route receives message
- AI SDK streams response with tool calls

### Step 2: Tool Interceptor Buffers Calls
- `textEditor` tool calls are intercepted
- Instead of executing immediately, calls are buffered
- Tool interceptor collects all file operations

### Step 3: Go Backend Creates Plan
- TypeScript calls `POST /api/plan/create-from-tools`
- Request body includes:
  - `workspaceId`
  - `chatMessageId`
  - `bufferedToolCalls[]` (file paths, operations, content)
- Go creates records in:
  - `workspace_plan` table
  - `workspace_plan_action_file` table
- Returns `planId`

### Step 4: Message Updated with Plan ID
- `responsePlanId` set on the chat message
- Centrifugo broadcasts plan update event

### Step 5: Frontend Renders Plan UI
- `handlePlanUpdatedAtom` receives Centrifugo event
- Frontend detects `responsePlanId` on message
- **Existing** `PlanChatMessage.tsx` renders:
  - Status badge (pending → planning → review)
  - Action file list with expand/collapse
  - Proceed/Ignore buttons
  - Nested chat input

### Step 6: User Action
- **Proceed**: Buffered tool calls are executed, files created/modified
- **Ignore**: Plan marked as ignored, no changes made
- **Follow-up**: User can ask questions before deciding

## Components to Build

| Component | Location | Purpose |
|-----------|----------|---------|
| Go endpoint | `pkg/api/handlers/plan-from-tools.go` | Creates plan records from buffered tool calls |
| Tool interceptor | `lib/ai/tools/interceptor.ts` | Buffers tool calls, calls Go endpoint |
| Proceed handler | `lib/ai/tools/execute-buffered.ts` | Executes buffered calls on approval |
| Message updater | `lib/ai/tools/plan-integration.ts` | Sets `responsePlanId` on messages |

## Reused Components (No Changes Needed)

- `components/PlanChatMessage.tsx` (~400 lines)
- `atoms/workspace.ts` - `handlePlanUpdatedAtom`
- Centrifugo plan update events
- `workspace_plan` and `workspace_plan_action_file` tables

## Key Implementation Notes

1. **Same database records** - Plans stored identically to legacy path
2. **Same UI component** - PlanChatMessage.tsx reused as-is
3. **Same Centrifugo events** - Existing real-time updates work
4. **Tool buffering** - Only file-modifying tools are buffered (textEditor, etc.)

---

## Implementation Decisions

### 1. Where to Buffer Tool Calls?

**Decision: Option C - Persist with plan creation**

| Approach | Pros | Cons |
|----------|------|------|
| A: In-memory (per-request) | Simple, no DB changes | Lost if request fails mid-stream |
| B: Persist to DB immediately | Survives failures, auditable | More complexity, extra DB writes |
| **C: Persist with plan creation** ✅ | Single DB transaction, clean | Slight delay before persisted |

**Flow:**
1. Tool calls buffered in memory during streaming request
2. When streaming completes (`onFinish`), call Go endpoint with all buffered calls
3. Go creates plan record AND stores buffered calls in `workspace_plan.buffered_tool_calls` (JSONB column) in single transaction
4. If request fails mid-stream, no orphaned data

### 2. How Does `responsePlanId` Get Set?

**Decision: Go backend handles it (existing logic)**

The Go endpoint already does this in `pkg/workspace/plan.go:303-304`:
```sql
UPDATE workspace_chat SET response_plan_id = $1 WHERE id = $2
```

**Flow:**
1. TypeScript calls Go `/api/plan/create-from-tools` with `chatMessageId`
2. Go creates plan record
3. Go updates `workspace_chat.response_plan_id` (existing logic)
4. Go publishes Centrifugo event (existing logic)
5. Frontend receives `chatmessage-updated` event with `responsePlanId`
6. `ChatMessage.tsx:159` detects `responsePlanId` and renders `PlanChatMessage`

**TypeScript doesn't need to set it** - Go backend already does this as part of plan creation.

### 3. Which Tools Trigger Plan Creation?

**Decision: Option A - Only `textEditor` with `create` or `str_replace`**

| Tool | Command | Triggers Plan? |
|------|---------|----------------|
| `textEditor` | `view` | ❌ No (read-only) |
| `textEditor` | `create` | ✅ Yes |
| `textEditor` | `str_replace` | ✅ Yes |
| `getChartContext` | - | ❌ No (read-only) |
| `latestSubchartVersion` | - | ❌ No (read-only) |
| `latestKubernetesVersion` | - | ❌ No (read-only) |
| `convertK8sToHelm` | - | ❌ No (different flow) |

**Interceptor Logic:**
```typescript
// In bufferedTools.ts
execute: async (params, { toolCallId }) => {
  // Read-only commands execute immediately
  if (params.command === "view") {
    return actuallyExecuteTool(params);
  }
  
  // Mutating commands get buffered
  if (params.command === "create" || params.command === "str_replace") {
    onToolCall({ toolCallId, toolName: "textEditor", args: params });
    return { success: true, buffered: true, message: `Pending: ${params.command} ${params.path}` };
  }
}
```

---

## Summary of Decisions

| Question | Recommendation |
|----------|----------------|
| Buffer location | In-memory during request, persist with plan creation (Option C) |
| `responsePlanId` | Go backend sets it (existing logic), Centrifugo notifies frontend |
| Which tools | Only `textEditor` with `create` or `str_replace` commands (Option A) |
| **Parity level** | **Option A: Accept Different AI Response Style** |

### Parity Decision: Option A

**What's identical:**
- Plan UI (`PlanChatMessage.tsx`) - same component
- Proceed/Ignore buttons - same functionality
- Database records - same tables and schema
- File operations - same execution path
- Centrifugo events - same real-time updates

**What's different:**
- AI's prose style - AI SDK responses may be more direct/concise vs Go worker's verbose plan descriptions
- Users get same functionality, slightly different wording

**Why Option A:**
- Simpler implementation - no message routing needed
- AI SDK maintains control of response generation
- Plan functionality is what matters, not exact wording

---

*Created: Dec 5, 2025*
*Updated: Dec 5, 2025 - Added implementation decisions*
*Related: PRDs/PR3.0_FEATURE_PARITY_IMPLEMENTATION_PLAN.md*

