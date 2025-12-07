---
date: 2025-12-07T20:30:31Z
researcher: mylessjs
git_commit: 9fd70641bd4d98973d351ec705dd952e653c45c1
branch: main
repository: UnchartedTerritory
topic: "Two-Phase Plan/Execute Architecture in Chartsmith AI SDK"
tags: [research, codebase, ai-sdk, plan-generation, tool-buffering, execution]
status: complete
last_updated: 2025-12-07
last_updated_by: mylessjs
---

# Research: Two-Phase Plan/Execute Architecture in Chartsmith AI SDK

**Date**: 2025-12-07T20:30:31Z
**Researcher**: mylessjs
**Git Commit**: 9fd70641bd4d98973d351ec705dd952e653c45c1
**Branch**: main
**Repository**: UnchartedTerritory

## Research Question

Document the current architecture for the two-phase plan/execute workflow, specifically:
1. How plans are generated (with and without buffered tool calls)
2. How file lists are determined before execution
3. The two execution paths: Path A (buffered tool calls) and Path B (text-only plans)

## Summary

The Chartsmith AI SDK implements a two-phase plan/execute workflow that separates plan creation from execution. The system currently has **two distinct execution paths**:

- **Path A (Buffered Tool Calls)**: When AI generates tool calls during streaming, they are buffered rather than executed. Execution happens when user clicks "Proceed" via `proceedPlanAction`.
- **Path B (Text-Only Plans)**: When plans are generated without tools enabled, the system uses `extractExpectedFilesViaLLM` to predict files, then executes via `executeViaAISDK` with a second LLM call.

The key architectural distinction is how each path determines the file list before execution.

## Detailed Findings

### 1. Entry Point: Chat API Route

**File**: `chartsmith/chartsmith-app/app/api/chat/route.ts`

The chat endpoint handles all incoming messages and routes them based on intent classification.

**Key Decision Points** (lines 185-265):
```typescript
switch (route.type) {
  case "off-topic":    // Decline without AI call
  case "plan":         // Plan generation - NO TOOLS
  case "proceed":      // Execution - tools enabled
  case "render":       // Acknowledge render request
  case "ai-sdk":       // Default conversational path
}
```

**Plan Route (lines 196-247)**:
- Tools are intentionally **disabled** to force descriptive text response
- Uses `CHARTSMITH_PLAN_SYSTEM_PROMPT` (without tool directives)
- Injects `getPlanOnlyUserMessage()` to instruct AI to describe without coding
- Creates plan via `createPlanFromToolCalls(authHeader, workspaceId, chatMessageId, [], text)`
- Empty array `[]` indicates text-only plan

**AI-SDK Route (lines 279-331)**:
- Tools are **enabled** via `createBufferedTools()`
- Tool calls are intercepted and pushed to `bufferedToolCalls` array
- On finish, creates plan via `createPlanFromToolCalls()` with populated tool calls
- Fallback: If `wasIntentPlan` but no tool calls, creates text-only plan

### 2. Intent Classification System

**Files**:
- `chartsmith/chartsmith-app/lib/ai/intent.ts`
- `chartsmith/pkg/api/handlers/intent.go`
- `chartsmith/pkg/llm/intent.go`

**Intent Structure**:
```typescript
interface Intent {
  isOffTopic: boolean;
  isPlan: boolean;
  isConversational: boolean;
  isChartDeveloper: boolean;
  isChartOperator: boolean;
  isProceed: boolean;
  isRender: boolean;
}
```

**Routing Logic** (`intent.ts:87-116`):
```typescript
// Priority order:
1. isProceed → { type: "proceed" }
2. isOffTopic && !isInitial && hasRevisions → { type: "off-topic" }
3. isRender → { type: "render" }
4. isPlan && !isConversational → { type: "plan" }   // KEY: Forces plan-only path
5. default → { type: "ai-sdk" }
```

**Critical Insight**: The condition `isPlan && !isConversational` routes to plan generation **without tools**, creating text-only plans.

### 3. Tool Buffering System

**Files**:
- `chartsmith/chartsmith-app/lib/ai/tools/bufferedTools.ts`
- `chartsmith/chartsmith-app/lib/ai/tools/toolInterceptor.ts`

**BufferedToolCall Interface** (`toolInterceptor.ts:12-17`):
```typescript
interface BufferedToolCall {
  id: string;           // Tool call ID from AI SDK
  toolName: string;     // "textEditor"
  args: Record<string, unknown>;  // {command, path, content?, oldStr?, newStr?}
  timestamp: number;    // Date.now()
}
```

**Buffering Logic** (`bufferedTools.ts:56-139`):
- `textEditor` with `view` command → **Execute immediately** (read-only)
- `textEditor` with `create` command → **Buffer for plan**
- `textEditor` with `str_replace` command → **Buffer for plan**
- All other tools → **Execute immediately** (read-only)

**Buffer Callback** (`route.ts:140-146`):
```typescript
const tools = workspaceId
  ? (chatMessageId
      ? createBufferedTools(authHeader, workspaceId, revisionNumber ?? 0, (toolCall) => {
          bufferedToolCalls.push(toolCall);  // Intercepts and stores
        }, chatMessageId)
      : createTools(...))  // Non-buffered for non-plan paths
  : undefined;
```

### 4. Plan Creation

**File**: `chartsmith/chartsmith-app/lib/ai/plan.ts`

**createPlanFromToolCalls Function** (lines 40-59):
```typescript
async function createPlanFromToolCalls(
  authHeader: string | undefined,
  workspaceId: string,
  chatMessageId: string,
  toolCalls: BufferedToolCall[],  // Empty for text-only plans
  description?: string            // Plan text for text-only plans
): Promise<string>
```

Calls Go endpoint `/api/plan/create-from-tools`:
- Creates plan record with status `'review'`
- Stores `buffered_tool_calls` in JSONB column
- Extracts action files from tool calls (or none for text-only)
- Sets `response_plan_id` on chat message
- Publishes Centrifugo events for real-time UI

### 5. Frontend Plan Display

**File**: `chartsmith/chartsmith-app/components/PlanChatMessage.tsx`

**handleProceed Function** (lines 155-195):
```typescript
const handleProceed = async () => {
  if (plan.bufferedToolCalls && plan.bufferedToolCalls.length > 0) {
    // PATH A: Execute buffered tool calls directly
    await proceedPlanAction(session, plan.id, wsId, revisionNumber);
  } else if (plan.description) {
    // PATH B: Execute text-only plan via AI SDK
    await executeViaAISDK({
      session,
      planId: plan.id,
      workspaceId: wsId,
      revisionNumber,
      planDescription: plan.description,
    });
  } else {
    // PATH C: Legacy Go worker fallback
    await createRevisionAction(session, plan.id);
  }
};
```

### 6. Path A: Buffered Tool Call Execution

**File**: `chartsmith/chartsmith-app/lib/workspace/actions/proceed-plan.ts`

**proceedPlanAction Function** (lines 87-242):

1. **Fetch buffered tool calls** from `workspace_plan.buffered_tool_calls`
2. **Validate** plan status is `'review'` and has tool calls
3. **Update status** to `'applying'`
4. **Execute each tool call** sequentially:
   - Mark file as `'creating'` (shows spinner)
   - Call Go `/api/tools/editor` endpoint
   - Mark file as `'created'` (shows checkmark)
5. **Update final status** to `'applied'` (or `'review'` if all failed)

**File List Source**: Derived from `buffered_tool_calls` array - exact match with what will be executed.

### 7. Path B: Text-Only Plan Execution

**File**: `chartsmith/chartsmith-app/lib/workspace/actions/execute-via-ai-sdk.ts`

**executeViaAISDK Function** (lines 201-329):

1. **Update status** to `'applying'`
2. **Extract expected files** via `extractExpectedFilesViaLLM()` (lines 32-91):
   ```typescript
   // Uses a SECOND LLM call to predict files from plan description
   const extractionPrompt = `You are a file path extractor.
     Given a plan description, identify ALL file paths...`;
   const result = await generateText({ model, messages: [...] });
   ```
3. **Mark files as `'pending'`** upfront in UI
4. **Execute via AI SDK** with tools enabled:
   - Uses `CHARTSMITH_EXECUTION_SYSTEM_PROMPT`
   - `onChunk` callback marks files as `'creating'`
   - `onStepFinish` callback marks files as `'created'`
5. **Update final status** to `'applied'`

**File List Source**: LLM prediction from plan text - **may differ** from actual tool calls.

### 8. Database Schema

**Table**: `workspace_plan`

| Column | Type | Description |
|--------|------|-------------|
| `id` | text | Primary key |
| `workspace_id` | text | Foreign key |
| `status` | text | 'pending', 'planning', 'review', 'applying', 'applied', 'ignored' |
| `description` | text | Plan text (for text-only plans) |
| `buffered_tool_calls` | jsonb | Array of `{id, toolName, args, timestamp}` |
| `proceed_at` | timestamp | When plan was executed |

**Table**: `workspace_plan_action_file`

| Column | Type | Description |
|--------|------|-------------|
| `plan_id` | text | FK to workspace_plan |
| `path` | text | File path |
| `action` | text | 'create', 'update' |
| `status` | text | 'pending', 'creating', 'created' |

### 9. Go Backend Endpoints

| Endpoint | Handler | Purpose |
|----------|---------|---------|
| `POST /api/plan/create-from-tools` | `plan.go:46-116` | Create plan with buffered tool calls |
| `POST /api/plan/publish-update` | `plan.go:378-399` | Send Centrifugo event for plan changes |
| `POST /api/plan/update-action-file-status` | `plan.go:258-342` | Update file status during execution |
| `POST /api/tools/editor` | `editor.go:75-118` | Execute file view/create/str_replace |

## Code References

### Plan Generation Flow
- `chartsmith/chartsmith-app/app/api/chat/route.ts:196-247` - Plan route handler (NO TOOLS)
- `chartsmith/chartsmith-app/lib/ai/prompts.ts:105-143` - `CHARTSMITH_PLAN_SYSTEM_PROMPT`
- `chartsmith/chartsmith-app/lib/ai/prompts.ts:152-155` - `getPlanOnlyUserMessage()`
- `chartsmith/chartsmith-app/lib/ai/plan.ts:40-59` - `createPlanFromToolCalls()`

### Intent Classification
- `chartsmith/chartsmith-app/lib/ai/intent.ts:42-59` - `classifyIntent()`
- `chartsmith/chartsmith-app/lib/ai/intent.ts:87-116` - `routeFromIntent()`
- `chartsmith/pkg/llm/intent.go:15-143` - Go intent classification via Groq

### Tool Buffering
- `chartsmith/chartsmith-app/lib/ai/tools/bufferedTools.ts:47-156` - `createBufferedTools()`
- `chartsmith/chartsmith-app/lib/ai/tools/toolInterceptor.ts:12-17` - `BufferedToolCall` interface
- `chartsmith/chartsmith-app/app/api/chat/route.ts:140-146` - Buffer callback setup

### Execution Paths
- `chartsmith/chartsmith-app/components/PlanChatMessage.tsx:165-180` - Path routing logic
- `chartsmith/chartsmith-app/lib/workspace/actions/proceed-plan.ts:87-242` - Path A execution
- `chartsmith/chartsmith-app/lib/workspace/actions/execute-via-ai-sdk.ts:201-329` - Path B execution
- `chartsmith/chartsmith-app/lib/workspace/actions/execute-via-ai-sdk.ts:32-91` - `extractExpectedFilesViaLLM()`

### Database
- `chartsmith/db/schema/tables/workspace-plan.yaml` - Plan table schema
- `chartsmith/db/schema/tables/workspace-plan-action-file.yaml` - Action files schema
- `chartsmith/pkg/workspace/types/types.go:59-87` - Go type definitions

### Go Backend
- `chartsmith/pkg/api/handlers/plan.go:46-116` - Create plan from tool calls
- `chartsmith/pkg/api/handlers/plan.go:258-342` - Update action file status
- `chartsmith/pkg/api/handlers/editor.go:75-118` - Text editor tool handler

## Architecture Documentation

### Two-Phase Workflow Comparison

| Aspect | Path A (Buffered) | Path B (Text-Only) |
|--------|-------------------|-------------------|
| **Plan Generation** | AI SDK with tools, buffered | AI SDK without tools |
| **Tool Calls** | Captured during streaming | None during planning |
| **File List Source** | From `bufferedToolCalls` array | LLM prediction via `extractExpectedFilesViaLLM()` |
| **Execution Trigger** | `proceedPlanAction()` | `executeViaAISDK()` |
| **LLM Calls for Execution** | 0 (uses stored calls) | 2 (prediction + execution) |
| **File List Accuracy** | Exact match guaranteed | May differ from execution |

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                       USER MESSAGE                               │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   INTENT CLASSIFICATION                          │
│    classifyIntent() → Go /api/intent/classify → Groq LLM        │
│    Returns: {isPlan, isConversational, isProceed, ...}          │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ROUTE DETERMINATION                            │
│    routeFromIntent() determines execution path                   │
│    isPlan && !isConversational → "plan" route                   │
│    default → "ai-sdk" route                                      │
└─────────────────────────────────────────────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
┌──────────────────────────┐     ┌──────────────────────────┐
│     "plan" ROUTE         │     │    "ai-sdk" ROUTE        │
│                          │     │                          │
│  streamText() NO TOOLS   │     │  streamText() + TOOLS    │
│  PLAN_SYSTEM_PROMPT      │     │  TOOL_SYSTEM_PROMPT      │
│  Descriptive text only   │     │  createBufferedTools()   │
│                          │     │  onToolCall → buffer[]   │
└──────────────────────────┘     └──────────────────────────┘
              │                                 │
              ▼                                 ▼
┌──────────────────────────┐     ┌──────────────────────────┐
│  createPlanFromToolCalls │     │  createPlanFromToolCalls │
│  toolCalls: []           │     │  toolCalls: [...]        │
│  description: <text>     │     │  (extracted from buffer) │
└──────────────────────────┘     └──────────────────────────┘
              │                                 │
              └────────────────┬────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PLAN IN 'review' STATUS                       │
│    workspace_plan table, buffered_tool_calls JSONB column       │
│    Frontend shows plan with "Proceed" button                     │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼ USER CLICKS PROCEED
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
┌──────────────────────────┐     ┌──────────────────────────┐
│  PATH A: Has buffered    │     │  PATH B: No buffered     │
│  tool calls              │     │  tool calls (text-only)  │
│                          │     │                          │
│  proceedPlanAction()     │     │  executeViaAISDK()       │
│  Execute stored calls    │     │  1. extractExpectedFiles │
│  File list: EXACT        │     │  2. streamText + tools   │
│                          │     │  File list: PREDICTED    │
└──────────────────────────┘     └──────────────────────────┘
              │                                 │
              └────────────────┬────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PLAN IN 'applied' STATUS                      │
│    Files created/modified, proceed_at timestamp set             │
└─────────────────────────────────────────────────────────────────┘
```

## Related Research

- `thoughts/shared/research/2025-12-06-chartsmith-ai-sdk-feature-parity.md`
- `thoughts/shared/research/2025-12-06-pr3-comprehensive-parity-gaps.md`
- `docs/plans/2025-12-06-two-phase-plan-execute-parity.md`

## Open Questions

1. **File List Mismatch Risk**: Path B's `extractExpectedFilesViaLLM()` uses a separate LLM call that may predict different files than what the execution LLM actually generates. What is the current impact on user experience?

2. **Plan Route Condition**: The condition `isPlan && !isConversational` determines whether tools are disabled. How often does intent classification correctly identify plan-worthy messages?

3. **Fallback Behavior**: When `extractExpectedFilesViaLLM()` fails, it falls back to regex extraction. What is the success rate of the regex fallback?
