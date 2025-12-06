# PR3.0 Feature Parity Implementation - Completion Report

**Status**: ✅ Complete
**Date**: December 6, 2025
**Branch**: `myles/vercel-ai-sdk-migration`

---

## Overview

PR3.0 implements full feature parity between the AI SDK chat path and the legacy Go worker path. The implementation follows **Option A** from the parity plan: same plan UI/flow with potentially different AI prose.

---

## Phase 1: Quick Wins ✅

### 1.1 Page Reload Guard (`components/WorkspaceContent.tsx`)

Added `beforeunload` event handler that warns users before leaving with:
- Uncommitted file changes (tracked via `allFilesWithContentPendingAtom`)
- Active streaming/rendering (tracked via `isRenderingAtom` and `currentStreamingMessageIdAtom`)
- Uses refs to ensure the event handler has access to current values

### 1.2 Rule-Based Followup Actions

**Files Modified:**
- `lib/chat/messageMapper.ts` - Added followup generation functions
- `hooks/useAISDKChatAdapter.ts` - Integrated followup generation
- `lib/workspace/actions/update-chat-message-response.ts` - Added followup persistence

**Implementation:**
- `hasFileModifyingToolCalls()` - Detects textEditor create/str_replace
- `generateFollowupActions()` - Adds "Render the chart" action after file changes
- Followup actions are persisted to database and displayed in ChatMessage UI

---

## Phase 2: Go Backend Integration ✅

### 2.0 Database Migration (`db/schema/tables/workspace-plan.yaml`)

Added `buffered_tool_calls` JSONB column:
```yaml
- name: buffered_tool_calls
  type: jsonb
  default: "'[]'::jsonb"
```

SchemaHero handles the migration automatically.

### 2.1 Intent Classification Endpoint (`pkg/api/handlers/intent.go`)

**Endpoint:** `POST /api/intent/classify`

**Request:**
```json
{
  "prompt": "string",
  "isInitialPrompt": boolean,
  "messageFromPersona": "auto" | "developer" | "operator"
}
```

**Response:**
```json
{
  "intent": {
    "isOffTopic": boolean,
    "isPlan": boolean,
    "isConversational": boolean,
    "isChartDeveloper": boolean,
    "isChartOperator": boolean,
    "isProceed": boolean,
    "isRender": boolean
  }
}
```

Uses existing `llm.GetChatMessageIntent()` for Groq-based classification.

### 2.2 TypeScript Intent Client (`lib/ai/intent.ts`)

- `classifyIntent()` - Calls Go endpoint
- `routeFromIntent()` - Determines routing (off-topic/proceed/render/ai-sdk)
- `OFF_TOPIC_DECLINE_MESSAGE` - Standard polite decline message

### 2.3 Intent Classification Integration (`app/api/chat/route.ts`)

The chat route now integrates full intent classification:

1. **Extract user prompt** from the last user message
2. **Call `classifyIntent()`** via Go/Groq endpoint
3. **Route based on intent**:
   - `off-topic` → Return `OFF_TOPIC_DECLINE_MESSAGE` immediately (no AI SDK call)
   - `proceed` → Return proceed indicator for adapter to handle
   - `render` → Return render indicator for adapter to handle
   - `ai-sdk` → Continue to AI SDK processing with buffered tools
4. **Fail-open**: If intent classification fails, proceed with AI SDK

### 2.4 Rollback Field Population (`lib/workspace/actions/commit-pending-changes.ts`)

Updated commit action to set `response_rollback_to_revision_number` on the first message of each revision, enabling the rollback link display in ChatMessage UI.

---

## Phase 3: Plan Workflow ✅

### 3.1 Tool Interceptor (`lib/ai/tools/toolInterceptor.ts`)

**Exports:**
- `BufferedToolCall` interface
- `ToolInterceptor` interface
- `createToolInterceptor()` factory
- `shouldBufferToolCall()` helper

```typescript
interface BufferedToolCall {
  id: string;
  toolName: string;
  args: Record<string, unknown>;
  timestamp: number;
}
```

### 3.2 Go Plan Creation Endpoint (`pkg/api/handlers/plan.go`)

**Endpoint:** `POST /api/plan/create-from-tools`

**Request:**
```json
{
  "workspaceId": "string",
  "chatMessageId": "string",
  "toolCalls": [
    {
      "id": "string",
      "toolName": "textEditor",
      "args": { "command": "create", "path": "...", "content": "..." },
      "timestamp": 1234567890
    }
  ]
}
```

**Response:**
```json
{
  "planId": "string"
}
```

**Actions:**
1. Creates `workspace_plan` record with `buffered_tool_calls` JSONB
2. Creates `workspace_plan_action_file` records for affected files
3. Sets `response_plan_id` on the chat message
4. Publishes Centrifugo `plan-updated` event

### 3.3 Buffered Tools (`lib/ai/tools/bufferedTools.ts`)

`createBufferedTools()` factory that wraps textEditor:
- `view` command: Executes immediately (read-only)
- `create` command: Buffered for plan workflow
- `str_replace` command: Buffered for plan workflow

Buffered calls return:
```json
{
  "success": true,
  "buffered": true,
  "message": "File change will be applied after review: create templates/deployment.yaml"
}
```

### 3.4 Chat Route Plan Integration (`app/api/chat/route.ts`)

The chat route now uses buffered tools and creates plans:

1. **Buffered tool collection**: When `chatMessageId` is provided, uses `createBufferedTools()` instead of `createTools()`
2. **Tool call buffer**: Collects `BufferedToolCall[]` during streaming via callback
3. **Plan creation on finish**: In `onFinish` callback, if buffered calls exist:
   ```typescript
   onFinish: async () => {
     if (bufferedToolCalls.length > 0 && workspaceId && chatMessageId) {
       await createPlanFromToolCalls(authHeader, workspaceId, chatMessageId, bufferedToolCalls);
     }
   }
   ```
4. **Error handling**: Plan creation failures are logged but don't fail the response

### 3.5 Adapter Updates (`hooks/useAISDKChatAdapter.ts`)

The adapter now passes `chatMessageId` to the route:

```typescript
const getChatBody = useCallback(
  (chatMessageId?: string) => ({
    provider, model, workspaceId, revisionNumber, persona,
    chatMessageId,  // PR3.0: Include for plan workflow
  }),
  [workspaceId, revisionNumber]
);
```

### 3.6 Proceed Action (`lib/workspace/actions/proceed-plan.ts`)

`proceedPlanAction()` server action:
1. Retrieves buffered tool calls from plan record
2. Validates plan is in "review" status
3. Updates status to "applying"
4. Executes each call via Go `/api/tools/editor` endpoint
5. Updates status to "applied"

Error handling: On failure, resets plan to "review" status.

---

## Phase 4: K8s Conversion Bridge ✅

### 4.1 Conversion Endpoint (`pkg/api/handlers/conversion.go`)

**Endpoint:** `POST /api/conversion/start`

**Request:**
```json
{
  "workspaceId": "string",
  "chatMessageId": "string",
  "sourceFiles": [
    { "filePath": "deployment.yaml", "fileContent": "..." }
  ]
}
```

**Response:**
```json
{
  "conversionId": "string"
}
```

**Actions:**
1. Creates `workspace_conversion` record
2. Creates `workspace_conversion_file` records
3. Sets `response_conversion_id` on chat message
4. Enqueues `new_conversion` work for Go pipeline

### 4.2 TypeScript Client (`lib/ai/conversion.ts`)

`startConversion()` function to trigger the conversion pipeline.

### 4.3 AI SDK Tool (`lib/ai/tools/convertK8s.ts`)

`createConvertK8sTool()` factory:
- Accepts array of K8s manifest files
- Triggers 6-step conversion pipeline
- Returns conversion ID for progress tracking

---

## Build Verification ✅

| Check | Status |
|-------|--------|
| TypeScript Lint | ✅ No ESLint warnings or errors |
| TypeScript Unit Tests | ✅ 94 passed |
| Go Build | ✅ Compiles successfully |
| Go Tests | ✅ All tests pass |

---

## Files Created

### TypeScript
| File | Purpose |
|------|---------|
| `lib/ai/intent.ts` | Intent classification client |
| `lib/ai/plan.ts` | Plan creation client |
| `lib/ai/conversion.ts` | Conversion client |
| `lib/ai/tools/toolInterceptor.ts` | Tool call buffering infrastructure |
| `lib/ai/tools/bufferedTools.ts` | Buffered tool implementations |
| `lib/ai/tools/convertK8s.ts` | K8s conversion AI SDK tool |
| `lib/workspace/actions/proceed-plan.ts` | Proceed action server action |

### Go
| File | Purpose |
|------|---------|
| `pkg/api/handlers/intent.go` | Intent classification endpoint |
| `pkg/api/handlers/plan.go` | Plan creation endpoint |
| `pkg/api/handlers/conversion.go` | Conversion bridge endpoint |

---

## Files Modified

| File | Changes |
|------|---------|
| `app/api/chat/route.ts` | **PR3.0 Integration**: Intent classification, buffered tools, plan creation on finish |
| `hooks/useAISDKChatAdapter.ts` | Pass `chatMessageId` to route for plan workflow |
| `components/WorkspaceContent.tsx` | Added page reload guard |
| `lib/chat/messageMapper.ts` | Added followup actions generation |
| `lib/workspace/actions/update-chat-message-response.ts` | Added followup actions parameter |
| `lib/workspace/actions/commit-pending-changes.ts` | Added rollback field population |
| `lib/ai/tools/index.ts` | Added new exports |
| `db/schema/tables/workspace-plan.yaml` | Added buffered_tool_calls column |
| `pkg/api/server.go` | Registered new endpoints |

---

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

---

## Manual Testing Checklist

- [ ] Page reload guard: Make file changes, try to close browser tab - warning appears
- [ ] Page reload guard: During streaming, try to close tab - warning appears
- [ ] Followup actions: After AI creates/edits a file, "Render the chart" button appears
- [ ] Intent classification: Send off-topic message - receive polite decline
- [ ] Intent classification: Send "proceed" - revision created
- [ ] Plan workflow: Ask to create a file - PlanChatMessage appears
- [ ] Plan workflow: Click Proceed - files are created
- [ ] Plan workflow: Click Ignore - plan marked ignored
- [ ] Rollback: After committing, rollback link appears on first message
- [ ] K8s conversion: Trigger conversion - ConversionProgress displays

---

*Last Updated: December 6, 2025*

