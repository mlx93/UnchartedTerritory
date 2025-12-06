# PR3.0: Final Feature Parity Implementation Plan

**Status**: Ready for Implementation
**Created**: 2025-12-05
**Research**: `docs/research/2025-12-05-MISSING-FEATURES-IMPLEMENTATION-NOTES.md`
**Depends On**: PR2.0 (AI SDK Reintegration - Complete)
**Estimated Effort**: 25-38 hours

---

## Overview

This plan implements the 6 remaining features needed for full parity between the AI SDK path and the legacy Go worker path. After PR2.0, the AI SDK transport is integrated but lacks these features:

1. **Plan Display UI** - Show `PlanChatMessage` with Proceed/Ignore workflow
2. **Rollback Functionality** - Enable rollback to previous revisions
3. **Intent Classification** - Pre-classify messages via Go/Groq before AI SDK
4. **K8s Conversion Bridge** - HTTP bridge to trigger Go conversion pipeline
5. **Followup Action Buttons** - Rule-based followup suggestions
6. **Page Reload Guard** - Warn before leaving with uncommitted changes

---

## Implementation Gaps Summary

Based on codebase analysis (December 5, 2025), these gaps were identified and are addressed by this plan:

| Gap | Description | Addressed In |
|-----|-------------|--------------|
| #1 | No `onFinish` callback in `/api/chat/route.ts` streamText() | Phase 2 (stub) → Phase 3 (full) |
| #2 | No `chatMessageId` in `ChatRequestBody` interface | Phase 2 |
| #3 | No intent classification before AI SDK | Phase 2 |
| #4 | `messageMapper.ts:161-164` notes `followupActions` and `responseRollbackToRevisionNumber` unsupported | Phase 1 (followup) + Phase 2 (rollback) |
| #5 | Go API missing endpoints: `/api/intent/classify`, `/api/plan/create-from-tools`, `/api/conversion/start` | Phase 2 + Phase 3 + Phase 4 |
| #6 | No `buffered_tool_calls` column in `workspace_plan` table | Phase 3 (migration first) |

---

## Current State Analysis

### What Exists (Post-PR2.0)

| Component | Status | Notes |
|-----------|--------|-------|
| AI SDK Chat | Working | `/api/chat` with streaming, tools |
| `PlanChatMessage.tsx` | Exists | Not triggered from AI SDK path |
| `RollbackModal.tsx` | Exists | Not wired to AI SDK messages |
| `ConversionProgress.tsx` | Exists | No way to trigger from AI SDK |
| Followup buttons in `ChatMessage.tsx` | Exists | `followupActions` never populated |
| `allFilesWithContentPendingAtom` | Exists | No `beforeunload` handler |

### What's Missing

| Feature | Gap |
|---------|-----|
| Plan Workflow | AI SDK writes directly to `content_pending`, no plan records created |
| Rollback | `responseRollbackToRevisionNumber` not set on AI SDK messages |
| Intent Classification | All messages go to AI SDK, no Groq pre-classification |
| K8s Conversion | No HTTP endpoint to trigger Go conversion pipeline |
| Followup Actions | `messageMapper.ts:162` explicitly notes "not supported" |
| Page Reload Guard | No `beforeunload` event handler exists |

---

## Desired End State

After this plan is complete:

1. **Plan Workflow**: When AI SDK calls `textEditor`, tool calls are buffered, a plan record is created, `PlanChatMessage` renders with Proceed/Ignore buttons, and Proceed executes the buffered changes.

2. **Rollback**: Messages that create new revisions have `responseRollbackToRevisionNumber` set, enabling the rollback link in `ChatMessage.tsx`.

3. **Intent Classification**: User messages are pre-classified via Go/Groq endpoint before AI SDK processes them. `IsOffTopic` messages receive a polite decline. `IsProceed` creates revisions directly. `IsRender` triggers rendering. Other intents proceed to AI SDK.

4. **K8s Conversion**: A TypeScript endpoint `/api/conversion/start` triggers the Go conversion pipeline, and `ConversionProgress.tsx` displays real-time status via existing Centrifugo events.

5. **Followup Actions**: After AI SDK responses complete, rule-based followup actions are added (e.g., "Render chart" after file changes).

6. **Page Reload Guard**: Browser warns users before leaving with uncommitted changes or during streaming.

### Verification

- All features work identically to legacy path
- Feature flag rollback still works (`NEXT_PUBLIC_USE_AI_SDK_CHAT=false`)
- Existing tests pass, new unit tests added for each feature

---

## What We're NOT Doing

- Changing the AI SDK system prompts or tool definitions
- Modifying Go worker queue processing logic
- Adding new database tables (using existing schema)
- Implementing percentage-based feature flag rollout
- Adding E2E tests (manual testing covers gaps)

---

## Implementation Approach

**Strategy**: Implement features in dependency order, with quick wins first.

```
Phase 1: Quick Wins (3-4 hours)
    ├── Page Reload Guard
    └── Followup Actions (rule-based)

Phase 2: Go Backend Integration (8-12 hours)
    ├── Intent Classification HTTP endpoint
    └── Rollback field population

Phase 3: Plan Workflow (10-14 hours)
    ├── Tool call interception
    ├── Plan creation via Go backend
    └── Plan display UI integration

Phase 4: K8s Conversion Bridge (4-8 hours)
    ├── HTTP bridge endpoint
    └── Frontend trigger integration
```

---

## Key Architectural Decisions

### Decision 1: Tool Call Buffering Location

**Question**: Where to buffer tool calls? In memory (per-request), or persist to DB immediately?

**Decision**: **Option C - In-memory during request, persist with plan creation**

**Flow**:
1. Tool calls are buffered in memory during the streaming request
2. When streaming completes (`onFinish`), call Go endpoint with all buffered calls
3. Go endpoint creates plan record AND stores buffered calls in `workspace_plan.buffered_tool_calls` (JSONB column) in a single transaction
4. If request fails mid-stream, no orphaned data

**Rationale**: Matches how the legacy path works - plans aren't created until the AI finishes "planning." Single transaction ensures atomicity.

### Decision 2: How `responsePlanId` Gets Set

**Question**: How does `responsePlanId` get set on the message?

**Decision**: **Go backend sets it (existing logic)**

**Flow**:
1. TypeScript calls Go `/api/plan/create-from-tools` with `chatMessageId`
2. Go creates plan record in `workspace_plan` table
3. Go updates `workspace_chat.response_plan_id` (existing logic at `pkg/workspace/plan.go:303-304`)
4. Go publishes Centrifugo `plan-updated` event (existing logic)
5. Frontend receives `chatmessage-updated` event with `responsePlanId`
6. `ChatMessage.tsx:159` detects `responsePlanId` and renders `PlanChatMessage`

**Rationale**: Reuses existing Go logic exactly. TypeScript doesn't need to set it - the backend already handles this as part of plan creation.

### Decision 3: Which Tools Trigger Plan Creation

**Question**: Which tools trigger plan creation? Only `textEditor`, or all file-modifying tools?

**Decision**: **Option A - Only `textEditor` with `create` or `str_replace` commands**

**Tool Behavior**:
| Tool | Command | Behavior |
|------|---------|----------|
| `textEditor` | `view` | Execute immediately (read-only, no side effects) |
| `textEditor` | `create` | Buffer for plan |
| `textEditor` | `str_replace` | Buffer for plan |
| `getChartContext` | - | Execute immediately (read-only) |
| `latestSubchartVersion` | - | Execute immediately (read-only) |
| `latestKubernetesVersion` | - | Execute immediately (read-only) |
| `convertK8sToHelm` | - | Execute immediately (triggers separate conversion pipeline) |

**Rationale**: Most conservative approach. Only file-mutating operations require user approval. Read-only operations execute immediately for better UX.

### Decision 4: Intent Classification Routing

**Question**: Should intent classification affect which AI model is used?

**Decision**: **No - Route all AI-bound intents to the same AI SDK handler**

**Routing Logic**:
| Intent | Action |
|--------|--------|
| `IsProceed` | Create revision from existing plan (Go handles) |
| `IsOffTopic` (+ not initial + has revision) | Return polite decline (no AI SDK call) |
| `IsRender` | Trigger render (Go handles) |
| All others (`IsPlan`, `IsConversational`, etc.) | Proceed to AI SDK |

**Rationale**: Simplicity. The AI SDK path handles planning via tool interception, so `IsPlan` vs `IsConversational` distinction is handled by whether the AI calls tools or not.

### Decision 5: Followup Actions Generation

**Question**: Should followup actions be AI-generated or rule-based?

**Decision**: **Option B - Rule-based**

**Rules**:
- If message contains `textEditor` tool call with `create` or `str_replace` → Add "Render the chart" followup
- Future: Could add more rules (e.g., "View diff" after str_replace)

**Rationale**: Simpler implementation, deterministic behavior, no additional AI calls needed.

### Decision 6: Rollback Field Population

**Question**: When should `responseRollbackToRevisionNumber` be set?

**Decision**: **Set on the first message that creates a new revision**

**Flow**:
1. User clicks Proceed on a plan → plan executes → files written to `content_pending`
2. User clicks Commit → `commitPendingChangesAction` creates new revision
3. In commit action, find the first chat message for the new revision
4. Set `response_rollback_to_revision_number = newRevisionNumber - 1` on that message
5. Centrifugo notifies frontend → rollback link appears

**Rationale**: Matches legacy behavior exactly. The rollback link appears on the first message of each revision, allowing users to roll back to the previous state.

---

## Phase 1: Quick Wins

### Overview
Implement low-complexity features that have no dependencies on other phases.

**Gaps Addressed in This Phase:**
- Gap #4 (partial): messageMapper.ts unsupported features → Enable `followupActions`
- No existing page reload guard → Add `beforeunload` handler

### Changes Required

#### 1.1 Page Reload Guard

**File**: `chartsmith-app/components/WorkspaceContent.tsx`

**Changes**: Add `useEffect` hook after line 88 (after existing hydration effect)

```typescript
// Add imports at top of file
import { allFilesWithContentPendingAtom, isRenderingAtom } from "@/atoms/workspace";

// Add after line 88 (after hydration useEffect)
useEffect(() => {
  const handleBeforeUnload = (e: BeforeUnloadEvent) => {
    // Check for pending file changes
    const filesWithPending = get(allFilesWithContentPendingAtom);
    const hasPending = filesWithPending.length > 0;

    // Check for active streaming/rendering
    const isRendering = get(isRenderingAtom);

    if (hasPending || isRendering) {
      e.preventDefault();
      e.returnValue = 'You have unsaved changes. Are you sure you want to leave?';
      return e.returnValue;
    }
  };

  window.addEventListener('beforeunload', handleBeforeUnload);
  return () => window.removeEventListener('beforeunload', handleBeforeUnload);
}, []);
```

**Note**: Need to use Jotai's `useAtomValue` or pass values as dependencies since `beforeunload` handler can't use hooks directly. Alternative implementation:

```typescript
// Better approach - track values in refs
const filesWithPendingRef = useRef<WorkspaceFile[]>([]);
const isStreamingRef = useRef(false);

const [filesWithPending] = useAtom(allFilesWithContentPendingAtom);
const [isRendering] = useAtom(isRenderingAtom);

// Keep refs in sync
useEffect(() => {
  filesWithPendingRef.current = filesWithPending;
}, [filesWithPending]);

useEffect(() => {
  isStreamingRef.current = isRendering;
}, [isRendering]);

useEffect(() => {
  const handleBeforeUnload = (e: BeforeUnloadEvent) => {
    const hasPending = filesWithPendingRef.current.length > 0;
    const isStreaming = isStreamingRef.current;

    if (hasPending || isStreaming) {
      e.preventDefault();
      e.returnValue = 'You have unsaved changes. Are you sure you want to leave?';
      return e.returnValue;
    }
  };

  window.addEventListener('beforeunload', handleBeforeUnload);
  return () => window.removeEventListener('beforeunload', handleBeforeUnload);
}, []);
```

#### 1.2 Followup Actions (Rule-Based)

**File**: `chartsmith-app/lib/chat/messageMapper.ts`

**Changes**: Add function to generate rule-based followup actions

```typescript
// Add after line 160 (where followupActions is noted as unsupported)

import { FollowupAction } from "@/lib/types/workspace";

/**
 * Generates rule-based followup actions based on message content and tool results
 */
export function generateFollowupActions(
  message: UIMessage,
  hasFileChanges: boolean
): FollowupAction[] {
  const actions: FollowupAction[] = [];

  // If message resulted in file changes, suggest rendering
  if (hasFileChanges) {
    actions.push({
      action: "render",
      label: "Render the chart"
    });
  }

  // Check for tool results that indicate specific actions
  const toolResults = message.parts?.filter(p => p.type === 'tool-result') ?? [];

  for (const result of toolResults) {
    if (result.toolName === 'textEditor') {
      const output = result.result as { success?: boolean; message?: string };
      if (output?.success && !actions.some(a => a.action === 'render')) {
        actions.push({
          action: "render",
          label: "Render the chart"
        });
      }
    }
  }

  return actions;
}

/**
 * Detects if a UIMessage contains file-modifying tool calls
 */
export function hasFileModifyingToolCalls(message: UIMessage): boolean {
  const toolInvocations = message.parts?.filter(p => p.type === 'tool-invocation') ?? [];

  return toolInvocations.some(invocation => {
    if (invocation.toolName === 'textEditor') {
      const args = invocation.args as { command?: string };
      return args?.command === 'create' || args?.command === 'str_replace';
    }
    return false;
  });
}
```

**File**: `chartsmith-app/hooks/useAISDKChatAdapter.ts`

**Changes**: Populate followupActions when persisting response

```typescript
// In the useEffect that persists responses (around line 180)
// After getting the response text, add:

import { generateFollowupActions, hasFileModifyingToolCalls } from "@/lib/chat/messageMapper";

// Inside the persistence useEffect:
const hasFileChanges = hasFileModifyingToolCalls(lastAssistant);
const followupActions = generateFollowupActions(lastAssistant, hasFileChanges);

// Pass to updateChatMessageResponseAction
updateChatMessageResponseAction(
  session,
  currentChatMessageId,
  textContent,
  true, // isIntentComplete
  followupActions // Add this parameter
);
```

**File**: `chartsmith-app/lib/workspace/actions/update-chat-message-response.ts`

**Changes**: Accept and persist followupActions parameter

```typescript
// Update function signature
export async function updateChatMessageResponseAction(
  session: Session,
  chatMessageId: string,
  response: string,
  isIntentComplete: boolean,
  followupActions?: FollowupAction[]
): Promise<void> {
  // ... existing code ...

  // Update SQL to include followup_actions
  await client.query(
    `UPDATE workspace_chat
     SET response = $1,
         is_intent_complete = $2,
         followup_actions = $3
     WHERE id = $4`,
    [response, isIntentComplete, JSON.stringify(followupActions ?? []), chatMessageId]
  );
}
```

### Success Criteria - Phase 1

#### Automated Verification:
- [ ] TypeScript compiles: `make -C chartsmith/chartsmith-app check`
- [ ] Unit tests pass: `make -C chartsmith/chartsmith-app test`
- [ ] Linting passes: `make -C chartsmith/chartsmith-app lint`

#### Manual Verification:
- [ ] Page reload guard: Make file changes, try to close browser tab - warning appears
- [ ] Page reload guard: During streaming, try to close tab - warning appears
- [ ] Followup actions: After AI creates/edits a file, "Render the chart" button appears
- [ ] Clicking followup action triggers the appropriate behavior

**Implementation Note**: After completing Phase 1 and all automated verification passes, pause for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Go Backend Integration

### Overview
Add HTTP endpoints to Go backend for intent classification and integrate rollback field population.

**Gaps Addressed in This Phase:**
- Gap #3: No intent classification in chat route → Add `/api/intent/classify` endpoint
- Gap #5 (partial): Go API server missing endpoints → Add `/api/intent/classify`
- Gap #4 (partial): messageMapper.ts unsupported features → Enable `responseRollbackToRevisionNumber`

**Preparation for Phase 3:**
- Add `chatMessageId` to `ChatRequestBody` now (Phase 3 depends on it)
- Add `onFinish` callback stub to `streamText()` (Phase 3 will expand it)

### Changes Required

#### 2.1 Intent Classification HTTP Endpoint

**File**: `chartsmith/pkg/api/handlers/intent.go` (NEW)

```go
package handlers

import (
	"encoding/json"
	"net/http"

	"github.com/replicatedhq/chartsmith/pkg/llm"
	"github.com/replicatedhq/chartsmith/pkg/workspace/types"
)

type ClassifyIntentRequest struct {
	Prompt            string `json:"prompt"`
	IsInitialPrompt   bool   `json:"isInitialPrompt"`
	MessageFromPersona string `json:"messageFromPersona"`
}

type ClassifyIntentResponse struct {
	Intent types.Intent `json:"intent"`
}

func ClassifyIntent(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	var req ClassifyIntentRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	intent, err := llm.GetChatMessageIntent(
		ctx,
		req.Prompt,
		req.IsInitialPrompt,
		req.MessageFromPersona,
	)
	if err != nil {
		http.Error(w, "failed to classify intent: "+err.Error(), http.StatusInternalServerError)
		return
	}

	resp := ClassifyIntentResponse{Intent: *intent}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(resp)
}
```

**File**: `chartsmith/pkg/api/server.go`

**Changes**: Register new endpoint

```go
// Add after line 23 (after existing tool endpoints)
mux.HandleFunc("/api/intent/classify", handlers.ClassifyIntent)
```

#### 2.2 TypeScript Intent Classification Client

**File**: `chartsmith-app/lib/ai/intent.ts` (NEW)

```typescript
import { callGoEndpoint } from "./tools/goEndpoint";

export interface Intent {
  isOffTopic: boolean;
  isPlan: boolean;
  isConversational: boolean;
  isChartDeveloper: boolean;
  isChartOperator: boolean;
  isProceed: boolean;
  isRender: boolean;
}

interface ClassifyIntentRequest {
  prompt: string;
  isInitialPrompt: boolean;
  messageFromPersona: string;
}

interface ClassifyIntentResponse {
  intent: Intent;
}

export async function classifyIntent(
  authHeader: string | undefined,
  prompt: string,
  isInitialPrompt: boolean,
  persona: string
): Promise<Intent> {
  const response = await callGoEndpoint<ClassifyIntentResponse>(
    "/api/intent/classify",
    {
      prompt,
      isInitialPrompt,
      messageFromPersona: persona,
    },
    authHeader
  );

  return response.intent;
}

/**
 * Determines how to route based on classified intent
 */
export type IntentRoute =
  | { type: "off-topic" }
  | { type: "proceed" }
  | { type: "render" }
  | { type: "ai-sdk" }; // Default - let AI SDK handle

export function routeFromIntent(intent: Intent, currentRevision: number): IntentRoute {
  // IsProceed - create revision from existing plan
  if (intent.isProceed) {
    return { type: "proceed" };
  }

  // IsOffTopic - polite decline (only if not initial message and has revision)
  if (intent.isOffTopic && !intent.isPlan && currentRevision > 0) {
    return { type: "off-topic" };
  }

  // IsRender - trigger render
  if (intent.isRender) {
    return { type: "render" };
  }

  // Default - let AI SDK process (covers IsPlan, IsConversational, etc.)
  return { type: "ai-sdk" };
}
```

#### 2.3 Integrate Intent Classification into Chat Route

**File**: `chartsmith-app/app/api/chat/route.ts`

**Changes**: Add intent classification before AI SDK processing

```typescript
// Add imports
import { classifyIntent, routeFromIntent } from "@/lib/ai/intent";
import { declineOffTopicAction } from "@/lib/workspace/actions/decline-off-topic";
import { createRevisionFromPlanAction } from "@/lib/workspace/actions/create-revision-from-plan";
import { enqueueRenderAction } from "@/lib/workspace/actions/enqueue-render";

// In POST handler, before streamText call (around line 140):

// Classify intent via Go/Groq
const lastUserMessage = messages.filter(m => m.role === 'user').pop();
if (lastUserMessage && workspaceId) {
  const userPrompt = lastUserMessage.content;
  const isInitialPrompt = revisionNumber === 0;

  try {
    const intent = await classifyIntent(
      authHeader,
      typeof userPrompt === 'string' ? userPrompt : JSON.stringify(userPrompt),
      isInitialPrompt,
      persona ?? 'auto'
    );

    const route = routeFromIntent(intent, revisionNumber ?? 0);

    switch (route.type) {
      case "off-topic":
        // Return polite decline without calling AI SDK
        return new Response(
          JSON.stringify({
            message: "I'm designed to help with Helm charts. Could you rephrase your question to be about Helm chart development or operations?"
          }),
          { status: 200, headers: { 'Content-Type': 'application/json' } }
        );

      case "proceed":
        // Create revision from existing plan
        await createRevisionFromPlanAction(session, workspaceId);
        return new Response(
          JSON.stringify({ message: "Created new revision from plan." }),
          { status: 200, headers: { 'Content-Type': 'application/json' } }
        );

      case "render":
        // Trigger render
        await enqueueRenderAction(session, workspaceId);
        return new Response(
          JSON.stringify({ message: "Render enqueued." }),
          { status: 200, headers: { 'Content-Type': 'application/json' } }
        );

      case "ai-sdk":
        // Continue to AI SDK processing below
        break;
    }
  } catch (err) {
    console.error('[/api/chat] Intent classification failed, proceeding with AI SDK:', err);
    // Fall through to AI SDK on intent classification failure
  }
}

// ... existing streamText call continues here
```

#### 2.4 Rollback Field Population

**File**: `chartsmith-app/lib/workspace/actions/commit-pending-changes.ts`

**Changes**: Set `responseRollbackToRevisionNumber` on the first message that created the revision

```typescript
// After creating the new revision, find and update the first message
// that initiated this revision

export async function commitPendingChangesAction(
  session: Session,
  workspaceId: string
): Promise<{ revisionNumber: number; firstMessageId?: string }> {
  // ... existing code to create revision ...

  const newRevisionNumber = /* result of creating revision */;

  // Find the first chat message for this revision and set rollback field
  const firstMessageResult = await client.query(
    `SELECT id FROM workspace_chat
     WHERE workspace_id = $1
     AND revision_number = $2
     ORDER BY created_at ASC
     LIMIT 1`,
    [workspaceId, newRevisionNumber]
  );

  if (firstMessageResult.rows.length > 0) {
    const firstMessageId = firstMessageResult.rows[0].id;

    // Set the rollback revision number (previous revision)
    await client.query(
      `UPDATE workspace_chat
       SET response_rollback_to_revision_number = $1
       WHERE id = $2`,
      [newRevisionNumber - 1, firstMessageId]
    );

    return { revisionNumber: newRevisionNumber, firstMessageId };
  }

  return { revisionNumber: newRevisionNumber };
}
```

**File**: `chartsmith-app/hooks/useAISDKChatAdapter.ts`

**Changes**: After commit succeeds, ensure the rollback field is reflected in local state

```typescript
// The Centrifugo event handler should pick this up automatically
// via handleChatMessageUpdated, but verify it's working
```

#### 2.5 Add `chatMessageId` to Chat Route (Preparation for Phase 3)

**File**: `chartsmith-app/app/api/chat/route.ts`

**Changes**: Add `chatMessageId` to request interface and extract it. This is needed for Phase 3 to associate plans with messages.

```typescript
// Update ChatRequestBody interface (around line 44-51):
interface ChatRequestBody {
  messages: UIMessage[];
  provider?: string;
  model?: string;
  workspaceId?: string;
  revisionNumber?: number;
  persona?: ChatPersona;
  chatMessageId?: string;  // PR3.0: Added for plan creation association
}

// In POST handler, extract chatMessageId:
const { messages, provider, model, workspaceId, revisionNumber, persona, chatMessageId } = body;

// Update logging:
console.log('[/api/chat] Request received:', {
  hasMessages: !!messages?.length,
  provider,
  model,
  workspaceId,
  revisionNumber,
  persona: persona ?? 'auto',
  chatMessageId,  // PR3.0
});
```

#### 2.6 Add `onFinish` Callback Stub (Preparation for Phase 3)

**File**: `chartsmith-app/app/api/chat/route.ts`

**Changes**: Add `onFinish` callback to `streamText()` call. In Phase 2 this is a stub; Phase 3 will expand it for plan creation.

```typescript
// Current streamText call (around line 138-144):
const result = streamText({
  model: modelInstance,
  system: systemPrompt,
  messages: convertToModelMessages(messages),
  tools,
  stopWhen: stepCountIs(5),
});

// Updated with onFinish stub:
const result = streamText({
  model: modelInstance,
  system: systemPrompt,
  messages: convertToModelMessages(messages),
  tools,
  stopWhen: stepCountIs(5),
  onFinish: async ({ response, finishReason, usage }) => {
    // PR3.0 Phase 2: Stub for future plan creation logic
    // Phase 3 will expand this to:
    // 1. Check for buffered tool calls
    // 2. Create plan via Go backend
    // 3. Associate plan with chatMessageId
    console.log('[/api/chat] Stream finished:', {
      finishReason,
      usage,
      chatMessageId,
      workspaceId,
    });
  },
});
```

**Note**: The `onFinish` callback receives `response`, `finishReason`, and `usage`. In Phase 3, we'll add the buffered tool call logic here.

### Success Criteria - Phase 2

#### Automated Verification:
- [ ] Go builds: `make -C chartsmith build`
- [ ] TypeScript compiles: `make -C chartsmith/chartsmith-app check`
- [ ] Go tests pass: `make -C chartsmith test`
- [ ] TypeScript tests pass: `make -C chartsmith/chartsmith-app test`

#### Manual Verification:
- [ ] Intent classification: Send off-topic message like "What's the weather?" - receive polite decline
- [ ] Intent classification: Send "proceed" or "yes, apply the changes" - revision created
- [ ] Intent classification: Send "render the chart" - render triggered
- [ ] Intent classification: Normal Helm questions route to AI SDK
- [ ] Rollback: After committing changes, rollback link appears on first message of revision
- [ ] Rollback: Clicking rollback link opens `RollbackModal` and works correctly

**Implementation Note**: After completing Phase 2 and all verification passes, pause for manual confirmation before proceeding to Phase 3.

---

## Phase 3: Plan Workflow

### Overview
Implement the full plan workflow: intercept tool calls, create plan records, display `PlanChatMessage`, and execute on Proceed.

**Gaps Addressed in This Phase:**
- Gap #1: No `onFinish` callback → Expand stub from Phase 2 to buffer tool calls and create plans
- Gap #2: No `chatMessageId` in chat route → Already added in Phase 2, now use it
- Gap #5 (partial): Go API server missing endpoints → Add `/api/plan/create-from-tools`
- Gap #6: No `buffered_tool_calls` column → Add database migration (FIRST)

**⚠️ This is the most complex phase. Follow the order carefully:**
1. Database migration (must exist before code that uses it)
2. Tool interception infrastructure
3. Go backend endpoint
4. TypeScript client
5. Chat route integration
6. Frontend integration

### Changes Required

#### 3.0 Database Migration (DO THIS FIRST)

**File**: `chartsmith/migrations/XXXXXX_add_buffered_tool_calls.sql` (NEW)

Create and run this migration BEFORE implementing any code that references `buffered_tool_calls`:

```sql
-- Add buffered_tool_calls column to store AI SDK tool calls for plan execution
ALTER TABLE workspace_plan
ADD COLUMN IF NOT EXISTS buffered_tool_calls JSONB DEFAULT '[]';

-- Add index for potential queries on buffered tool calls
CREATE INDEX IF NOT EXISTS idx_workspace_plan_has_buffered_tools
ON workspace_plan ((buffered_tool_calls != '[]'::jsonb));

COMMENT ON COLUMN workspace_plan.buffered_tool_calls IS
'Stores AI SDK tool calls that are buffered during streaming and executed on Proceed. Format: [{id, toolName, args, timestamp}]';
```

**Rollback Migration**: `chartsmith/migrations/XXXXXX_add_buffered_tool_calls_down.sql`

```sql
ALTER TABLE workspace_plan DROP COLUMN IF EXISTS buffered_tool_calls;
```

**Verification**: After running migration, verify column exists:
```bash
make -C chartsmith db-migrate
# Then verify:
psql -c "SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'workspace_plan' AND column_name = 'buffered_tool_calls';"
```

#### 3.1 Tool Call Interception Infrastructure

**File**: `chartsmith-app/lib/ai/tools/toolInterceptor.ts` (NEW)

```typescript
import { ToolCall, ToolResult } from "ai";

export interface BufferedToolCall {
  id: string;
  toolName: string;
  args: Record<string, unknown>;
  timestamp: number;
}

export interface ToolInterceptor {
  buffer: BufferedToolCall[];
  shouldBuffer: boolean;

  intercept(toolCall: ToolCall): BufferedToolCall;
  clear(): void;
  getBufferedCalls(): BufferedToolCall[];
}

export function createToolInterceptor(): ToolInterceptor {
  let buffer: BufferedToolCall[] = [];
  let shouldBuffer = true; // Always buffer for plan workflow

  return {
    get buffer() { return buffer; },
    get shouldBuffer() { return shouldBuffer; },

    intercept(toolCall: ToolCall): BufferedToolCall {
      const buffered: BufferedToolCall = {
        id: toolCall.toolCallId,
        toolName: toolCall.toolName,
        args: toolCall.args as Record<string, unknown>,
        timestamp: Date.now(),
      };

      if (shouldBuffer) {
        buffer.push(buffered);
      }

      return buffered;
    },

    clear() {
      buffer = [];
    },

    getBufferedCalls() {
      return [...buffer];
    },
  };
}
```

#### 3.2 Plan Creation via Go Backend

**File**: `chartsmith/pkg/api/handlers/plan.go` (NEW)

```go
package handlers

import (
	"encoding/json"
	"net/http"

	"github.com/replicatedhq/chartsmith/pkg/workspace"
)

type CreatePlanFromToolCallsRequest struct {
	WorkspaceID   string     `json:"workspaceId"`
	ChatMessageID string     `json:"chatMessageId"`
	ToolCalls     []ToolCall `json:"toolCalls"`
}

type ToolCall struct {
	ID       string                 `json:"id"`
	ToolName string                 `json:"toolName"`
	Args     map[string]interface{} `json:"args"`
}

type CreatePlanFromToolCallsResponse struct {
	PlanID string `json:"planId"`
}

func CreatePlanFromToolCalls(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	var req CreatePlanFromToolCallsRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	// Generate plan description from tool calls
	description := generatePlanDescription(req.ToolCalls)

	// Create plan record
	plan, err := workspace.CreatePlan(ctx, req.ChatMessageID, req.WorkspaceID, false)
	if err != nil {
		http.Error(w, "failed to create plan: "+err.Error(), http.StatusInternalServerError)
		return
	}

	// Update plan with description and action files
	actionFiles := extractActionFiles(req.ToolCalls)
	err = workspace.UpdatePlanWithActions(ctx, plan.ID, description, actionFiles)
	if err != nil {
		http.Error(w, "failed to update plan: "+err.Error(), http.StatusInternalServerError)
		return
	}

	// Set plan status to "review"
	err = workspace.UpdatePlanStatus(ctx, plan.ID, "review")
	if err != nil {
		http.Error(w, "failed to update plan status: "+err.Error(), http.StatusInternalServerError)
		return
	}

	resp := CreatePlanFromToolCallsResponse{PlanID: plan.ID}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(resp)
}

func generatePlanDescription(toolCalls []ToolCall) string {
	// Generate markdown description of what the plan will do
	var description string
	for _, tc := range toolCalls {
		if tc.ToolName == "textEditor" {
			command := tc.Args["command"].(string)
			path := tc.Args["path"].(string)
			switch command {
			case "create":
				description += "- Create file: `" + path + "`\n"
			case "str_replace":
				description += "- Modify file: `" + path + "`\n"
			}
		}
	}
	return description
}

func extractActionFiles(toolCalls []ToolCall) []workspace.ActionFileInput {
	var files []workspace.ActionFileInput
	seen := make(map[string]bool)

	for _, tc := range toolCalls {
		if tc.ToolName == "textEditor" {
			path := tc.Args["path"].(string)
			command := tc.Args["command"].(string)

			if seen[path] {
				continue
			}
			seen[path] = true

			action := "update"
			if command == "create" {
				action = "create"
			}

			files = append(files, workspace.ActionFileInput{
				Path:   path,
				Action: action,
			})
		}
	}
	return files
}
```

**File**: `chartsmith/pkg/api/server.go`

**Changes**: Register plan endpoint

```go
mux.HandleFunc("/api/plan/create-from-tools", handlers.CreatePlanFromToolCalls)
```

#### 3.3 TypeScript Plan Creation Client

**File**: `chartsmith-app/lib/ai/plan.ts` (NEW)

```typescript
import { callGoEndpoint } from "./tools/goEndpoint";
import { BufferedToolCall } from "./tools/toolInterceptor";

interface CreatePlanRequest {
  workspaceId: string;
  chatMessageId: string;
  toolCalls: BufferedToolCall[];
}

interface CreatePlanResponse {
  planId: string;
}

export async function createPlanFromToolCalls(
  authHeader: string | undefined,
  workspaceId: string,
  chatMessageId: string,
  toolCalls: BufferedToolCall[]
): Promise<string> {
  const response = await callGoEndpoint<CreatePlanResponse>(
    "/api/plan/create-from-tools",
    {
      workspaceId,
      chatMessageId,
      toolCalls,
    },
    authHeader
  );

  return response.planId;
}
```

#### 3.4 Modified Chat Route for Plan Workflow

**File**: `chartsmith-app/app/api/chat/route.ts`

**Changes**: Intercept tool calls and create plan instead of executing directly

```typescript
// This is a significant change to the streaming flow
// Key insight: We need to intercept tool calls DURING streaming,
// not after. This requires modifying how streamText handles tools.

import { createToolInterceptor, BufferedToolCall } from "@/lib/ai/tools/toolInterceptor";
import { createPlanFromToolCalls } from "@/lib/ai/plan";

// In POST handler:

const toolInterceptor = createToolInterceptor();
const bufferedToolCalls: BufferedToolCall[] = [];

// Create modified tools that buffer instead of execute
const bufferedTools = workspaceId ? createBufferedTools(
  authHeader,
  workspaceId,
  revisionNumber ?? 0,
  (toolCall) => {
    bufferedToolCalls.push(toolInterceptor.intercept(toolCall));
  }
) : undefined;

const result = streamText({
  model: modelInstance,
  system: systemPrompt,
  messages: convertToModelMessages(messages),
  tools: bufferedTools,
  maxSteps: 5,
  onFinish: async ({ response }) => {
    // After streaming completes, if we have buffered tool calls, create a plan
    if (bufferedToolCalls.length > 0 && workspaceId && chatMessageId) {
      try {
        const planId = await createPlanFromToolCalls(
          authHeader,
          workspaceId,
          chatMessageId,
          bufferedToolCalls
        );

        // The plan creation updates the message's response_plan_id
        // Centrifugo will notify the frontend
        console.log('[/api/chat] Created plan:', planId);
      } catch (err) {
        console.error('[/api/chat] Failed to create plan:', err);
      }
    }
  },
});

return result.toUIMessageStreamResponse();
```

#### 3.5 Buffered Tools Implementation

**File**: `chartsmith-app/lib/ai/tools/bufferedTools.ts` (NEW)

Per **Decision 3**, only `textEditor` with `create` or `str_replace` commands triggers buffering. All other tools execute immediately.

```typescript
import { tool, ToolCall } from "ai";
import { z } from "zod";
import { callGoEndpoint } from "./goEndpoint";
import { createGetChartContextTool } from "./getChartContext";
import { createLatestSubchartVersionTool } from "./latestSubchartVersion";
import { createLatestKubernetesVersionTool } from "./latestKubernetesVersion";

type ToolCallCallback = (toolCall: ToolCall) => void;

export function createBufferedTools(
  authHeader: string | undefined,
  workspaceId: string,
  revisionNumber: number,
  onToolCall: ToolCallCallback
) {
  return {
    // textEditor: Only buffer create/str_replace, execute view immediately
    textEditor: tool({
      description: `A text editor tool for viewing, creating, and editing files.
Commands:
- view: Read file contents
- create: Create a new file with specified content
- str_replace: Replace text in an existing file`,
      parameters: z.object({
        command: z.enum(["view", "create", "str_replace"]),
        path: z.string().describe("File path relative to chart root"),
        content: z.string().optional().describe("File content for create command"),
        oldStr: z.string().optional().describe("Text to replace for str_replace"),
        newStr: z.string().optional().describe("Replacement text for str_replace"),
      }),
      execute: async (params, { toolCallId }) => {
        // VIEW: Execute immediately (read-only, no side effects)
        if (params.command === "view") {
          return callGoEndpoint("/api/tools/editor", {
            command: params.command,
            workspaceId,
            path: params.path,
            revisionNumber,
          }, authHeader);
        }

        // CREATE/STR_REPLACE: Buffer for plan workflow
        onToolCall({
          toolCallId,
          toolName: "textEditor",
          args: params,
        } as ToolCall);

        // Return a "pending" result so AI knows the action was acknowledged
        return {
          success: true,
          message: `File change will be applied after review: ${params.command} ${params.path}`,
          buffered: true,
        };
      },
    }),

    // All other tools execute immediately (read-only, no buffering needed)
    getChartContext: createGetChartContextTool(authHeader, workspaceId, revisionNumber),
    latestSubchartVersion: createLatestSubchartVersionTool(authHeader),
    latestKubernetesVersion: createLatestKubernetesVersionTool(authHeader),
  };
}
```

#### 3.6 Execute Buffered Tool Calls on Proceed

**File**: `chartsmith-app/lib/workspace/actions/proceed-plan.ts` (NEW)

```typescript
"use server";

import { Session } from "next-auth";
import { getPool } from "@/lib/db/pool";
import { callGoEndpoint } from "@/lib/ai/tools/goEndpoint";

interface BufferedToolCall {
  id: string;
  toolName: string;
  args: Record<string, unknown>;
}

export async function proceedPlanAction(
  session: Session,
  planId: string,
  workspaceId: string,
  revisionNumber: number
): Promise<void> {
  const client = await getPool().connect();

  try {
    // Get the plan and its associated chat message
    const planResult = await client.query(
      `SELECT chat_message_ids FROM workspace_plan WHERE id = $1`,
      [planId]
    );

    if (planResult.rows.length === 0) {
      throw new Error("Plan not found");
    }

    const chatMessageIds = planResult.rows[0].chat_message_ids;

    // Get buffered tool calls from the message metadata
    // (We need to store these when creating the plan)
    const toolCallsResult = await client.query(
      `SELECT buffered_tool_calls FROM workspace_plan WHERE id = $1`,
      [planId]
    );

    const bufferedToolCalls: BufferedToolCall[] =
      JSON.parse(toolCallsResult.rows[0].buffered_tool_calls || "[]");

    // Execute each buffered tool call
    for (const toolCall of bufferedToolCalls) {
      if (toolCall.toolName === "textEditor") {
        await callGoEndpoint("/api/tools/editor", {
          command: toolCall.args.command,
          workspaceId,
          path: toolCall.args.path,
          content: toolCall.args.content,
          oldStr: toolCall.args.oldStr,
          newStr: toolCall.args.newStr,
          revisionNumber,
        }, session.user?.authHeader);
      }
    }

    // Update plan status to "applying"
    await client.query(
      `UPDATE workspace_plan SET status = 'applying' WHERE id = $1`,
      [planId]
    );

    // The commit flow will be triggered separately by the user
    // after reviewing the pending changes

  } finally {
    client.release();
  }
}
```

#### 3.7 Frontend Plan Display Integration

**File**: `chartsmith-app/hooks/useAISDKChatAdapter.ts`

**Changes**: Handle `responsePlanId` in message mapping

```typescript
// In mapUIMessagesToMessages or the adapter logic:
// When a message has responsePlanId (set by Centrifugo event after plan creation),
// the existing ChatMessage.tsx:159-176 will automatically render PlanChatMessage

// The key is ensuring the message mapper preserves responsePlanId from DB
// This should already work via the existing messageMapper.ts
```

**File**: `chartsmith-app/lib/chat/messageMapper.ts`

**Changes**: Ensure `responsePlanId` is mapped

```typescript
// In mapUIMessageToMessage function, add:
export function mapUIMessageToMessage(
  uiMessage: UIMessage,
  dbMessage?: Partial<Message> // Add optional DB message for merging
): Message {
  return {
    // ... existing mapping ...
    responsePlanId: dbMessage?.responsePlanId, // Preserve from DB
    // ...
  };
}
```

### Success Criteria - Phase 3

#### Automated Verification:
- [ ] Go builds: `make -C chartsmith build`
- [ ] TypeScript compiles: `make -C chartsmith/chartsmith-app check`
- [ ] All tests pass: `make -C chartsmith test && make -C chartsmith/chartsmith-app test`

#### Manual Verification:
- [ ] Send message asking to create a file - `PlanChatMessage` appears instead of direct file creation
- [ ] Plan shows correct file list and action types (create/update)
- [ ] Plan status shows "review" with Proceed/Ignore buttons
- [ ] Click Proceed - files are created/modified, plan status changes to "applying" then "applied"
- [ ] Click Ignore - plan status changes to "ignored", no file changes
- [ ] Nested chat input in plan works for asking follow-up questions

**Implementation Note**: This is the most complex phase. After completing Phase 3 and all verification passes, pause for thorough manual testing before proceeding to Phase 4.

---

## Phase 4: K8s Conversion Bridge

### Overview
Create HTTP bridge endpoint to trigger the existing Go conversion pipeline from the AI SDK path.

**Gaps Addressed in This Phase:**
- Gap #5 (final): Go API server missing endpoints → Add `/api/conversion/start`
- No way to trigger conversion from AI SDK path → Add `convertK8sToHelm` tool

### Changes Required

#### 4.1 Conversion HTTP Endpoint

**File**: `chartsmith/pkg/api/handlers/conversion.go` (NEW)

```go
package handlers

import (
	"encoding/json"
	"net/http"

	"github.com/replicatedhq/chartsmith/pkg/persistence"
	"github.com/replicatedhq/chartsmith/pkg/workspace"
)

type StartConversionRequest struct {
	WorkspaceID   string           `json:"workspaceId"`
	ChatMessageID string           `json:"chatMessageId"`
	SourceFiles   []ConversionFile `json:"sourceFiles"`
}

type ConversionFile struct {
	FilePath    string `json:"filePath"`
	FileContent string `json:"fileContent"`
}

type StartConversionResponse struct {
	ConversionID string `json:"conversionId"`
}

func StartConversion(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	var req StartConversionRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	// Create conversion record
	conversion, err := workspace.CreateConversion(
		ctx,
		req.WorkspaceID,
		req.ChatMessageID,
		convertFiles(req.SourceFiles),
	)
	if err != nil {
		http.Error(w, "failed to create conversion: "+err.Error(), http.StatusInternalServerError)
		return
	}

	// Enqueue conversion work
	err = persistence.EnqueueWork(ctx, "new_conversion", map[string]string{
		"conversionId": conversion.ID,
	})
	if err != nil {
		http.Error(w, "failed to enqueue conversion: "+err.Error(), http.StatusInternalServerError)
		return
	}

	resp := StartConversionResponse{ConversionID: conversion.ID}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(resp)
}

func convertFiles(files []ConversionFile) []workspace.ConversionFileInput {
	result := make([]workspace.ConversionFileInput, len(files))
	for i, f := range files {
		result[i] = workspace.ConversionFileInput{
			FilePath:    f.FilePath,
			FileContent: f.FileContent,
		}
	}
	return result
}
```

**File**: `chartsmith/pkg/api/server.go`

**Changes**: Register conversion endpoint

```go
mux.HandleFunc("/api/conversion/start", handlers.StartConversion)
```

#### 4.2 TypeScript Conversion Client

**File**: `chartsmith-app/lib/ai/conversion.ts` (NEW)

```typescript
import { callGoEndpoint } from "./tools/goEndpoint";

interface ConversionFile {
  filePath: string;
  fileContent: string;
}

interface StartConversionRequest {
  workspaceId: string;
  chatMessageId: string;
  sourceFiles: ConversionFile[];
}

interface StartConversionResponse {
  conversionId: string;
}

export async function startConversion(
  authHeader: string | undefined,
  workspaceId: string,
  chatMessageId: string,
  sourceFiles: ConversionFile[]
): Promise<string> {
  const response = await callGoEndpoint<StartConversionResponse>(
    "/api/conversion/start",
    {
      workspaceId,
      chatMessageId,
      sourceFiles,
    },
    authHeader
  );

  return response.conversionId;
}
```

#### 4.3 AI SDK Tool for K8s Conversion

**File**: `chartsmith-app/lib/ai/tools/convertK8s.ts` (NEW)

```typescript
import { tool } from "ai";
import { z } from "zod";
import { startConversion } from "../conversion";

export function createConvertK8sTool(
  authHeader: string | undefined,
  workspaceId: string,
  chatMessageId: string
) {
  return tool({
    description: `Convert Kubernetes manifests to a Helm chart.
Provide an array of K8s manifest files with their paths and YAML content.
The conversion will analyze, template, and package them into a Helm chart.`,
    parameters: z.object({
      files: z.array(z.object({
        filePath: z.string().describe("Original file path (e.g., 'deployment.yaml')"),
        fileContent: z.string().describe("Full YAML content of the Kubernetes manifest"),
      })).describe("Array of K8s manifest files to convert"),
    }),
    execute: async ({ files }) => {
      try {
        const conversionId = await startConversion(
          authHeader,
          workspaceId,
          chatMessageId,
          files.map(f => ({
            filePath: f.filePath,
            fileContent: f.fileContent,
          }))
        );

        return {
          success: true,
          conversionId,
          message: `Conversion started. Track progress with conversion ID: ${conversionId}`,
        };
      } catch (error) {
        return {
          success: false,
          message: `Failed to start conversion: ${error}`,
        };
      }
    },
  });
}
```

**File**: `chartsmith-app/lib/ai/tools/index.ts`

**Changes**: Add conversion tool to createTools

```typescript
import { createConvertK8sTool } from "./convertK8s";

export function createTools(
  authHeader: string | undefined,
  workspaceId: string,
  revisionNumber: number,
  chatMessageId?: string
) {
  return {
    getChartContext: createGetChartContextTool(authHeader, workspaceId, revisionNumber),
    textEditor: createTextEditorTool(authHeader, workspaceId, revisionNumber),
    latestSubchartVersion: createLatestSubchartVersionTool(authHeader),
    latestKubernetesVersion: createLatestKubernetesVersionTool(authHeader),
    // Add conversion tool if chatMessageId is provided
    ...(chatMessageId ? {
      convertK8sToHelm: createConvertK8sTool(authHeader, workspaceId, chatMessageId),
    } : {}),
  };
}
```

#### 4.4 Update Chat Route to Pass chatMessageId

**File**: `chartsmith-app/app/api/chat/route.ts`

**Changes**: Pass chatMessageId to createTools

```typescript
// When creating tools, pass the chatMessageId so conversion tool can use it
const tools = workspaceId
  ? createTools(authHeader, workspaceId, revisionNumber ?? 0, chatMessageId)
  : undefined;
```

### Success Criteria - Phase 4

#### Automated Verification:
- [ ] Go builds: `make -C chartsmith build`
- [ ] TypeScript compiles: `make -C chartsmith/chartsmith-app check`
- [ ] All tests pass

#### Manual Verification:
- [ ] Ask AI to convert K8s manifests to Helm - conversion starts
- [ ] `ConversionProgress.tsx` displays with real-time status updates
- [ ] All 6 conversion steps complete successfully
- [ ] Resulting Helm chart files appear in the workspace

**Implementation Note**: After completing Phase 4 and all verification passes, the full feature parity implementation is complete.

---

## Testing Strategy

### Unit Tests

**New test files needed:**
- `lib/ai/__tests__/intent.test.ts` - Intent classification client
- `lib/ai/__tests__/plan.test.ts` - Plan creation client
- `lib/ai/__tests__/conversion.test.ts` - Conversion client
- `lib/ai/tools/__tests__/toolInterceptor.test.ts` - Tool interception
- `lib/ai/tools/__tests__/bufferedTools.test.ts` - Buffered tools
- `lib/chat/__tests__/messageMapper.test.ts` - Add followupActions tests

### Integration Tests

**Scenarios to test:**
- Intent classification → off-topic decline
- Intent classification → proceed creates revision
- Intent classification → render triggers render
- Plan creation from buffered tool calls
- Plan proceed executes changes
- Plan ignore marks as ignored
- Conversion start triggers Go pipeline

### Manual Testing Steps

1. **Page Reload Guard**
   - Make file changes (uncommitted)
   - Try to close tab - confirm warning appears
   - During AI streaming, try to close tab - confirm warning

2. **Followup Actions**
   - Ask AI to create a file
   - Verify "Render the chart" button appears after completion
   - Click button, verify render triggers

3. **Intent Classification**
   - "What's the weather?" → polite decline
   - "proceed" / "yes apply" → revision created
   - "render" → render triggered
   - "add a values.yaml file" → AI SDK processes normally

4. **Plan Workflow**
   - "Create a deployment.yaml file" → Plan appears
   - Review file list in plan
   - Click Proceed → file created
   - Or Click Ignore → plan marked ignored, no file

5. **Rollback**
   - Create multiple revisions via Proceed
   - Find first message of each revision
   - Click rollback link → RollbackModal appears
   - Confirm rollback → workspace reverts

6. **K8s Conversion**
   - Upload K8s manifests or paste content
   - Ask AI to convert to Helm
   - Watch ConversionProgress through 6 steps
   - Verify resulting Helm chart

---

## Performance Considerations

### Intent Classification Latency
- Groq call adds ~300-500ms before AI SDK
- Acceptable for strict parity
- Could optimize by caching common intents (future enhancement)

### Plan Creation Overhead
- Additional Go API call after streaming completes
- Centrifugo event propagation for UI update
- Should be < 200ms total

### Conversion Pipeline
- Uses existing Go workers (battle-tested)
- 6-step pipeline can take 1-2 minutes
- Real-time progress updates via Centrifugo

---

## Migration Notes

### Database Changes

No new tables required. Adding one column to existing table.

**Note**: The migration for `buffered_tool_calls` is documented in **Phase 3, Section 3.0** and should be run as the first step of Phase 3 implementation.

### Feature Flag
All changes are additive to the AI SDK path. Feature flag `NEXT_PUBLIC_USE_AI_SDK_CHAT=false` still reverts to legacy path.

### Rollback Procedure
If issues occur:
1. Set `NEXT_PUBLIC_USE_AI_SDK_CHAT=false`
2. Redeploy
3. Users immediately use legacy Go worker path

---

## Implementation Considerations

### 1. AI Understanding of Buffered Tool Results

When `textEditor` returns `buffered: true`, the AI needs to understand that the action is queued, not failed.

**Solution**: The tool response message is explicit:
```typescript
return {
  success: true,
  message: `File change will be applied after review: ${params.command} ${params.path}`,
  buffered: true,
};
```

The `success: true` tells the AI the operation was acknowledged. The message explains it's pending review. The AI should naturally continue with its response explaining what it proposed.

**If issues arise**, add system prompt guidance:
```
When a tool returns "buffered: true", the file change has been queued for user review.
Do not retry the operation. Continue explaining what you proposed.
```

### 2. Centrifugo Event Timing

The plan relies on Centrifugo to notify the frontend when `responsePlanId` is set. If there's latency:

**Primary Path** (should work):
1. Go sets `response_plan_id` on message
2. Go publishes Centrifugo `chatmessage-updated` event
3. Frontend receives event, updates message, renders `PlanChatMessage`

**Fallback** (if Centrifugo is slow):
- The `useAISDKChatAdapter` already polls/refetches messages
- Add optimistic update: After `onFinish` calls plan creation, immediately fetch the updated message

```typescript
// In onFinish callback after createPlanFromToolCalls succeeds:
const planId = await createPlanFromToolCalls(...);

// Optimistic: Immediately update local message state
setMessages(prev => prev.map(m =>
  m.id === chatMessageId
    ? { ...m, responsePlanId: planId }
    : m
));
```

### 3. Error Handling Strategy

| Scenario | Handling |
|----------|----------|
| Intent classification fails | Fail-open to AI SDK (logged, not blocking) |
| Plan creation fails | Log error, message persists without plan (user can retry) |
| Proceed execution fails | Show error toast, plan stays in "review" state |
| Centrifugo disconnected | UI still works via polling/refetch |

### 4. Race Condition: Multiple Tool Calls

If AI makes multiple `textEditor` calls in one response:
- All are buffered in order
- Single plan created with all changes
- Proceed executes all in order

The `bufferedToolCalls` array preserves order via timestamps.

---

## References

- Research: `docs/research/2025-12-05-MISSING-FEATURES-IMPLEMENTATION-NOTES.md`
- PR2.0 Tech PRD: `PRDs/PR2.0_REINTEGRATION_Tech_PRD.md`
- Feature Inventory: `PRDs/PR1.9_FULL_PARITY_FEATURE_INVENTORY.md`

### Key File Locations

| Component | File |
|-----------|------|
| PlanChatMessage | `chartsmith-app/components/PlanChatMessage.tsx` |
| ChatMessage | `chartsmith-app/components/ChatMessage.tsx` |
| RollbackModal | `chartsmith-app/components/RollbackModal.tsx` |
| ConversionProgress | `chartsmith-app/components/ConversionProgress.tsx` |
| Intent classification (Go) | `chartsmith/pkg/llm/intent.go` |
| Plan creation (Go) | `chartsmith/pkg/workspace/plan.go` |
| Conversion pipeline (Go) | `chartsmith/pkg/listener/new-conversion.go` |
| AI SDK chat route | `chartsmith-app/app/api/chat/route.ts` |
| Message mapper | `chartsmith-app/lib/chat/messageMapper.ts` |
| AI SDK adapter | `chartsmith-app/hooks/useAISDKChatAdapter.ts` |

---

*This implementation plan provides the complete path to 100% feature parity between the AI SDK and legacy paths.*

*Last Updated: December 5, 2025*
