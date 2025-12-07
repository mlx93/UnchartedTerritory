# AI SDK Migration: Fix Duplicate Text & Replace Go LLM Calls

## Overview

This plan addresses two interconnected issues in the ChartSmith AI SDK migration:

1. **Immediate Issue**: Plan text appears twice in the UI (streaming response + "Proposed Plan" card)
2. **Architectural Issue**: Go backend still makes direct Anthropic API calls for plan execution

The solution routes all LLM calls through the TypeScript AI SDK layer while keeping Go as a file system executor only.

## Current State Analysis

### Architecture (Myles Branch)

```
USER PROMPT → AI SDK (Plan Phase, no tools) → Plan text stored
                                                    ↓
                                           User clicks "Proceed"
                                                    ↓
            ┌──────────────────────────────────────┴──────────────────────────────────────┐
            │                                                                              │
    bufferedToolCalls.length > 0?                                     bufferedToolCalls.length === 0?
            ↓                                                                              ↓
    proceedPlanAction()                                               createRevisionAction()
    Execute stored tool calls                                         Enqueue to Go worker
    via Go /api/tools/editor                                          Go makes Anthropic calls
            ↓                                                                              ↓
    [WORKING - AI SDK path]                                           [LEGACY - violates requirement]
```

### Key Discoveries

| File | Line | Finding |
|------|------|---------|
| `chartsmith-app/app/api/chat/route.ts:189-240` | Plan generation uses `CHARTSMITH_PLAN_SYSTEM_PROMPT` with NO tools |
| `chartsmith-app/app/api/chat/route.ts:221-228` | Text-only plans created with empty `bufferedToolCalls` array |
| `chartsmith-app/components/PlanChatMessage.tsx:255-258` | Displays `plan.description` via ReactMarkdown (causes duplicate) |
| `chartsmith-app/components/PlanChatMessage.tsx:259-320` | Shows `actionFiles` list only when status is "applying" or "applied" |
| `chartsmith-app/components/PlanChatMessage.tsx:164-168` | AI SDK path requires `bufferedToolCalls.length > 0` |
| `chartsmith-app/lib/workspace/actions/proceed-plan.ts:122-123` | Throws error when `bufferedToolCalls.length === 0` |
| `pkg/llm/system.go:89-98` | Go `detailedPlanSystemPrompt` generates `<chartsmithActionPlan>` XML tags |
| `pkg/llm/parser.go:105-136` | Regex parser extracts file list from XML tags |
| `pkg/listener/execute-plan.go:105` | `CreateExecutePlan()` generates XML file list after user clicks Proceed |

### Root Cause of Duplicate Text

1. **Chat stream**: AI SDK streams plan response → rendered as chat message
2. **PlanChatMessage**: Same text stored in `plan.description` → rendered again in Proposed Plan card
3. **Main branch difference**: Card shows `actionFiles` list, not `description` text

### Root Cause of Empty bufferedToolCalls

1. Plan phase uses NO tools (`route.ts:207`)
2. `createPlanFromToolCalls` called with empty array (`route.ts:227`)
3. `proceedPlanAction` rejects empty tool calls (`proceed-plan.ts:122-123`)
4. Falls through to legacy Go worker path

## Desired End State

### Architecture (After Migration)

```
USER PROMPT → AI SDK (Plan Phase, no tools) → Plan text stored
                                                    ↓
                                           User clicks "Proceed"
                                                    ↓
                           AI SDK (Execution Phase, WITH tools)
                                                    ↓
                              textEditor tool calls Go /api/tools/editor
                                                    ↓
                              Files created/modified, streamed to UI
```

### Verification Criteria

1. **No duplicate text**: Proposed Plan card shows file list, not plan description
2. **All LLM calls via AI SDK**: Go backend only handles file I/O
3. **Feature parity**: 13+ files created for "nginx deployment" (matching main branch)
4. **Real-time updates**: File changes stream to UI as they're made
5. **Provider switching**: Execution phase supports same providers as plan phase

## What We're NOT Doing

- NOT implementing `<chartsmithActionPlan>` XML generation (Go-specific pattern)
- NOT modifying the Go worker queue system (will be deprecated)
- NOT changing Centrifugo infrastructure (still used for realtime)
- NOT changing database schema (reusing existing columns)
- NOT implementing embedding-based file selection (future enhancement)

## Implementation Approach

The key insight is that text-only plans need a **second AI SDK call** to generate file operations when the user clicks Proceed. This mirrors the main branch's two-phase approach but routes everything through AI SDK.

---

## Phase 1: Fix Duplicate Text Display

### Overview

Modify `PlanChatMessage` to NOT duplicate the plan description. The plan text streams as a chat message - the "Proposed Plan" card should show a placeholder indicating that file changes will be generated when the user clicks Proceed.

This matches the main branch behavior where:
1. Plan text streams to chat
2. User sees "Proposed Plan" card with action buttons
3. File list only appears AFTER clicking Proceed (during execution)

### Changes Required

#### 1.1 Update PlanChatMessage Rendering Logic

**File**: `chartsmith-app/components/PlanChatMessage.tsx`

**Current behavior** (lines 229-258):
```tsx
<div className="markdown-content">
  <ReactMarkdown>{plan.description}</ReactMarkdown>
</div>
```

**New behavior**: Don't show description in the card at all. Show a placeholder message for "review" status, or file list during/after execution.

```tsx
{/* For review status - show placeholder (no duplicate of streaming text) */}
{plan.status === 'review' && (!plan.actionFiles || plan.actionFiles.length === 0) && (
  <div className={`text-sm italic ${theme === "dark" ? "text-gray-400" : "text-gray-500"}`}>
    Click Proceed to generate file changes based on the plan above.
  </div>
)}

{/* For applying/applied - show file list (handled below) */}
{/* Description is already visible in the chat stream - no need to duplicate */}
```

#### 1.2 Keep ActionFiles Display for Execution States

**File**: `chartsmith-app/components/PlanChatMessage.tsx`

The existing condition is correct - file list appears when status is 'applying' or 'applied':

```tsx
{(plan.status === 'applying' || plan.status === 'applied') && (
```

This matches the main branch behavior - file list only shows during/after execution.

### Success Criteria

#### Automated Verification:
- [x] TypeScript compiles: `cd chartsmith-app && npm run build`
- [x] Linting passes: `cd chartsmith-app && npm run lint`

#### Manual Verification:
- [ ] Plan card shows "Click Proceed to generate file changes" placeholder
- [ ] No duplicate plan text visible in UI (text only appears in chat stream)
- [ ] Streaming chat message shows full plan description

---

## Phase 2: AI SDK Execution Path for Text-Only Plans

### Overview

When the user clicks Proceed on a text-only plan (no buffered tool calls), trigger a new AI SDK call with the textEditor tool enabled. This mirrors the Go main branch's two-phase approach:

1. **Plan phase**: Text description only (no file list)
2. **Execution phase**: AI generates file operations, file list populated dynamically as files are created

The key difference from the previous approach: we do NOT extract the file list upfront. Instead, the file list is built dynamically during execution as the AI calls the textEditor tool - exactly like Go's `CreateExecutePlan` function in `pkg/llm/execute-plan.go`.

### Changes Required

#### 2.1 Add Execution Prompt

**File**: `chartsmith-app/lib/ai/prompts.ts`

```typescript
/**
 * Execution system prompt - WITH TOOL DIRECTIVES
 *
 * Used during Phase 2 (execution) when user clicks Proceed.
 * Instructs the AI to execute the plan using the textEditor tool.
 * Mirrors Go: executePlanSystemPrompt
 */
export const CHARTSMITH_EXECUTION_SYSTEM_PROMPT = `You are ChartSmith, an expert AI assistant specializing in the creation and modification of Helm charts.

You have access to the following tool:

- **textEditor**: A file editing tool with three commands:
  - \`view\`: View the contents of a file. Use this before editing.
  - \`create\`: Create a new file with the specified content.
  - \`str_replace\`: Replace a specific string in a file with new content.

## Execution Instructions

You are executing a previously approved plan. Your task is to implement all file changes described in the plan.

1. For each file in the plan:
   - Use \`view\` first to see current contents (if updating)
   - Use \`create\` for new files
   - Use \`str_replace\` for modifying existing files

2. Work through files systematically:
   - Start with Chart.yaml and values.yaml
   - Then process templates/_helpers.tpl
   - Finally process other template files

3. Create complete, production-ready content:
   - Include proper Helm templating ({{ .Values.* }})
   - Follow Kubernetes best practices
   - Include configurable values for all important settings

4. Do NOT:
   - Explain what you're doing (just execute)
   - Ask for confirmation (plan is already approved)
   - Skip any files mentioned in the plan

## Code Formatting

- Use 2 spaces for indentation in all YAML files
- Ensure valid YAML and Helm template syntax
- Use proper Helm templating expressions where appropriate`;

/**
 * Get execution instruction based on plan type
 */
export function getExecutionInstruction(planDescription: string, isInitialChart: boolean): string {
  const verb = isInitialChart ? 'create' : 'update';
  return `Execute the following plan to ${verb} the Helm chart. Implement ALL file changes described:

${planDescription}

Begin execution now. Create or modify each file using the textEditor tool.`;
}
```

#### 2.2 Create Execute Via AI SDK Server Action

**File**: `chartsmith-app/lib/workspace/actions/execute-via-ai-sdk.ts` (new file)

```typescript
"use server";

import { streamText, stepCountIs } from 'ai';
import { Session } from "@/lib/types/session";
import { getDB } from "@/lib/data/db";
import { getParam } from "@/lib/data/param";
import { getModel } from "@/lib/ai";
import { createTools } from "@/lib/ai/tools";
import {
  CHARTSMITH_EXECUTION_SYSTEM_PROMPT,
  getExecutionInstruction,
} from "@/lib/ai/prompts";
import { callGoEndpoint } from "@/lib/ai/tools/utils";

interface ExecuteViaAISDKParams {
  session: Session;
  planId: string;
  workspaceId: string;
  revisionNumber: number;
  planDescription: string;
  provider?: string;
}

/**
 * Publishes a plan update event via Go backend
 */
async function publishPlanUpdate(
  workspaceId: string,
  planId: string
): Promise<void> {
  try {
    await callGoEndpoint<{ success: boolean }>(
      "/api/plan/publish-update",
      { workspaceId, planId }
    );
  } catch (error) {
    console.error("[executeViaAISDK] Failed to publish plan update:", error);
  }
}

/**
 * Adds or updates an action file in the plan.
 * This mimics Go's behavior in pkg/listener/execute-plan.go:116-153
 * where action files are added dynamically as they're discovered.
 *
 * Uses the existing /api/plan/update-action-file-status endpoint,
 * which we extend to support adding new files (not just updating existing ones).
 */
async function addOrUpdateActionFile(
  workspaceId: string,
  planId: string,
  path: string,
  action: "create" | "update",
  status: "pending" | "creating" | "created"
): Promise<void> {
  try {
    await callGoEndpoint<{ success: boolean }>(
      "/api/plan/update-action-file-status",  // Existing endpoint, extended to add files
      { workspaceId, planId, path, action, status }
    );
  } catch (error) {
    console.error(`[executeViaAISDK] Failed to add/update action file ${path}:`, error);
  }
}

/**
 * Executes a text-only plan via AI SDK with tools enabled.
 *
 * This is the AI SDK execution path for plans that were created without
 * buffered tool calls (i.e., plans generated during the plan-only phase).
 *
 * This mirrors Go's two-phase approach in pkg/listener/execute-plan.go:
 * - File list is NOT known upfront
 * - As the AI calls textEditor, we add files to actionFiles dynamically
 * - UI shows "selecting files..." initially, then file list streams in
 *
 * Flow:
 * 1. Update plan status to 'applying'
 * 2. Call AI SDK with execution prompt + textEditor tool
 * 3. As tools are called, add action files dynamically (like Go does)
 * 4. Tools execute against Go /api/tools/editor endpoint
 * 5. Update plan status to 'applied' on success, 'review' on failure
 */
export async function executeViaAISDK({
  session,
  planId,
  workspaceId,
  revisionNumber,
  planDescription,
  provider,
}: ExecuteViaAISDKParams): Promise<void> {
  if (!session?.user?.id) {
    throw new Error("Unauthorized");
  }

  const db = getDB(await getParam("DB_URI"));

  // 1. Update plan status to 'applying'
  await db.query(
    `UPDATE workspace_plan SET status = 'applying', updated_at = NOW() WHERE id = $1`,
    [planId]
  );
  await publishPlanUpdate(workspaceId, planId);

  // 2. Prepare execution context
  const isInitialChart = revisionNumber === 0;
  const executionInstruction = getExecutionInstruction(planDescription, isInitialChart);

  const model = getModel(provider);
  const tools = createTools(undefined, workspaceId, revisionNumber);

  // Track files being processed
  const processedFiles = new Set<string>();

  try {
    // 3. Execute via AI SDK with tools
    const result = await streamText({
      model,
      system: CHARTSMITH_EXECUTION_SYSTEM_PROMPT,
      messages: [{ role: 'user', content: executionInstruction }],
      tools,
      stopWhen: stepCountIs(50), // Allow many tool calls for multi-file operations (AI SDK v5 pattern)
      onStepFinish: async ({ toolCalls }) => {
        // Add/update action files as tools are called
        // This mimics Go's behavior where file list is built dynamically
        for (const toolCall of toolCalls ?? []) {
          if (toolCall.toolName === 'textEditor') {
            const args = toolCall.args as { path?: string; command?: string };
            if (args.path && !processedFiles.has(args.path)) {
              const action = args.command === 'create' ? 'create' : 'update';

              if (args.command === 'view') {
                // First view - add file with 'pending' status
                await addOrUpdateActionFile(workspaceId, planId, args.path, action, 'pending');
              } else if (args.command === 'create' || args.command === 'str_replace') {
                // File created/updated - mark as created
                await addOrUpdateActionFile(workspaceId, planId, args.path, action, 'created');
                processedFiles.add(args.path);
              }
            }
          }
        }
      },
    });

    // Consume the stream to completion
    for await (const _ of result.textStream) {
      // Stream consumed - side effects handled in onStepFinish
    }

    // 4. Update plan status to 'applied'
    await db.query(
      `UPDATE workspace_plan SET status = 'applied', proceed_at = NOW(), updated_at = NOW() WHERE id = $1`,
      [planId]
    );
    await publishPlanUpdate(workspaceId, planId);

  } catch (error) {
    console.error('[executeViaAISDK] Execution failed:', error);

    // Reset to review on failure (matches Go behavior)
    await db.query(
      `UPDATE workspace_plan SET status = 'review', updated_at = NOW() WHERE id = $1`,
      [planId]
    );
    await publishPlanUpdate(workspaceId, planId);

    throw error;
  }
}
```

#### 2.3 Update PlanChatMessage to Use AI SDK Execution

**File**: `chartsmith-app/components/PlanChatMessage.tsx`

Modify `handleProceed` (around line 154-182):

```typescript
import { executeViaAISDK } from "@/lib/workspace/actions/execute-via-ai-sdk";

const handleProceed = async () => {
  if (!session || !plan) return;

  const wsId = workspaceId || plan.workspaceId;
  if (!wsId) return;

  setIsProceeding(true);
  try {
    if (plan.bufferedToolCalls && plan.bufferedToolCalls.length > 0) {
      // Path A: AI SDK plans with buffered tool calls
      // Execute stored tool calls directly
      const revisionNumber = workspaceToUse?.currentRevisionNumber ?? 0;
      await proceedPlanAction(session, plan.id, wsId, revisionNumber);
    } else if (plan.description) {
      // Path B: Text-only plans (NEW)
      // Execute via AI SDK with tools enabled
      // File list will be built dynamically during execution (like Go does)
      const revisionNumber = workspaceToUse?.currentRevisionNumber ?? 0;
      await executeViaAISDK({
        session,
        planId: plan.id,
        workspaceId: wsId,
        revisionNumber,
        planDescription: plan.description,
      });
    } else {
      // Path C: Legacy fallback
      const updatedWorkspace = await createRevisionAction(session, plan.id);
      if (updatedWorkspace && setWorkspace) {
        setWorkspace(updatedWorkspace);
      }
    }
    onProceed?.();
  } catch (error) {
    console.error('[PlanChatMessage] Proceed failed:', error);
  } finally {
    setIsProceeding(false);
  }
};
```

#### 2.4 Extend Existing UpdateActionFileStatus Endpoint

**File**: `pkg/api/handlers/plan.go`

The existing `UpdateActionFileStatus` endpoint (lines 248-331) returns an error if the file is not found. Extend it to **add the file if not found** instead of returning an error. This allows TypeScript to add action files dynamically during execution.

**Current behavior** (lines 309-312):
```go
if !found {
    writeBadRequest(w, "Action file not found: "+req.Path)
    return
}
```

**New behavior** - add the file if not found:
```go
if !found {
    // Add new action file (supports dynamic file list building during AI SDK execution)
    plan.ActionFiles = append(plan.ActionFiles, types.ActionFile{
        Path:   req.Path,
        Action: req.Action, // Need to add Action field to request
        Status: req.Status,
    })
}
```

**Also update the request struct** (line 249) to include `Action`:
```go
// UpdateActionFileStatusRequest represents a request to update or add an action file
type UpdateActionFileStatusRequest struct {
    WorkspaceID string `json:"workspaceId"`
    PlanID      string `json:"planId"`
    Path        string `json:"path"`
    Action      string `json:"action,omitempty"` // "create" or "update" - required when adding new file
    Status      string `json:"status"`           // "pending", "creating", "created"
}
```

**Update validation** (around line 266) to require `Action` when adding new files:
```go
// Validate required fields
if req.WorkspaceID == "" || req.PlanID == "" || req.Path == "" || req.Status == "" {
    writeBadRequest(w, "workspaceId, planId, path, and status are required")
    return
}
```

This approach:
- Reuses the existing endpoint instead of creating a new one
- Maintains backward compatibility (existing callers that only update status still work)
- No new route registration needed - `/api/plan/update-action-file-status` already exists

**Update TypeScript caller** in `execute-via-ai-sdk.ts` to use existing endpoint:
```typescript
async function addOrUpdateActionFile(
  workspaceId: string,
  planId: string,
  path: string,
  action: "create" | "update",
  status: "pending" | "creating" | "created"
): Promise<void> {
  try {
    await callGoEndpoint<{ success: boolean }>(
      "/api/plan/update-action-file-status",  // Use existing endpoint
      { workspaceId, planId, path, action, status }
    );
  } catch (error) {
    console.error(`[executeViaAISDK] Failed to add/update action file ${path}:`, error);
  }
}
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `cd chartsmith-app && npm run build`
- [ ] Go compiles: `cd chartsmith && go build ./...`
- [ ] Linting passes: `cd chartsmith-app && npm run lint`

#### Manual Verification:
- [ ] Click Proceed on text-only plan → files start being created
- [ ] File status updates in real-time (spinner → checkmark)
- [ ] 10+ files created for "nginx deployment"
- [ ] Files match main branch quality (deployment, service, ingress, etc.)

---

## Phase 3: Deprecate Go LLM Calls

### Overview

Mark Go LLM functions as deprecated and add documentation. Do NOT remove them yet - they serve as fallback.

### Changes Required

#### 3.1 Add Deprecation Comments to Go LLM Functions

**File**: `pkg/llm/execute-plan.go`

```go
// Deprecated: CreateExecutePlan is part of the legacy Go-based execution path.
// New plans should use the AI SDK execution path via TypeScript.
// This function will be removed once AI SDK execution is stable.
func CreateExecutePlan(...) {
```

**File**: `pkg/llm/initial-plan.go`, `pkg/llm/plan.go`, `pkg/llm/execute-action.go`

Add similar deprecation comments.

#### 3.2 Add Migration Documentation

**File**: `docs/ARCHITECTURE.md` (update existing)

Add section documenting the new AI SDK execution path:

```markdown
## AI SDK Migration Status

### Current Architecture (Hybrid)

- **Plan Generation**: AI SDK (TypeScript)
- **Plan Execution**: AI SDK (TypeScript) for new plans, Go for legacy plans

### Execution Paths

1. **AI SDK Path** (recommended):
   - Plans created via `/api/chat` route
   - Execution via `executeViaAISDK()` server action
   - Go backend only handles file I/O via `/api/tools/editor`

2. **Legacy Go Path** (deprecated):
   - Plans created via Go queue workers
   - Execution via `execute_plan` → `apply_plan` queue jobs
   - Go makes direct Anthropic API calls

### Migration Progress

- [x] Plan generation via AI SDK
- [x] Tool execution via Go `/api/tools/editor`
- [x] AI SDK execution path for text-only plans
- [ ] Remove legacy Go LLM code (future)
```

### Success Criteria

#### Automated Verification:
- [ ] Go compiles with deprecation comments: `cd chartsmith && go build ./...`

#### Manual Verification:
- [ ] Legacy path still works as fallback
- [ ] Documentation accurately describes architecture

---

## Testing Strategy

### Unit Tests

- Intent routing: Verify `routeFromIntent` returns correct types
- Tool buffering: Verify callbacks collect tool calls
- Action file updates: Verify dynamic file list building

### Integration Tests

- End-to-end plan generation → placeholder display
- End-to-end plan execution via AI SDK with dynamic file list
- Real-time UI updates during execution

### Manual Testing Steps

1. Start fresh workspace
2. Enter "create a simple nginx deployment"
3. Verify plan streams to chat (no duplicate in card)
4. Verify Proposed Plan card shows "Click Proceed to generate file changes"
5. Click "Proceed"
6. Verify "selecting files..." appears initially
7. Verify files appear sequentially with status updates as they're created
8. Verify 10+ files created (Chart.yaml, values.yaml, templates/*)
9. Verify files appear in file explorer and editor
10. Accept changes and verify chart is valid

## Performance Considerations

- No upfront file list extraction - files discovered during execution (matches Go behavior)
- AI SDK execution may be slightly slower than Go (additional HTTP round-trips)
- No regression in streaming performance expected
- Dynamic action file updates add minimal overhead (one DB write per file)

## Migration Notes

- **Backward Compatibility**: Legacy Go path remains functional as fallback
- **No Database Migrations**: Reuses existing `action_files` column
- **Feature Flag**: None needed - routing based on `bufferedToolCalls` presence
- **Rollback**: If issues arise, plans fall through to legacy Go path

## References

- Implementation Plan: `/docs/plans/2025-12-06-two-phase-plan-execute-parity.md`
- Migration Status: `/docs/plans/2025-12-06-ai-sdk-migration-next-steps.md`
- Requirements: `/Replicated_Chartsmith.md`
- Main Branch Test: `/docs/testing/MAIN_BRANCH_NGINX_DEPLOYMENT_TEST.md`

---

## Summary

This plan addresses both immediate (duplicate text) and architectural (Go LLM calls) issues through:

1. **Phase 1**: Fix UI to show placeholder instead of duplicate description
2. **Phase 2**: Route execution through AI SDK with textEditor tool, building file list dynamically (mimics Go's two-phase approach)
3. **Phase 3**: Deprecate (don't remove) Go LLM code

The result is a clean architecture where:
- All LLM calls go through TypeScript AI SDK
- Go backend is a file system executor only
- Feature parity with main branch is maintained (same two-phase flow)
- File list is built dynamically during execution (like Go)
- Legacy fallback remains available
