# Two-Phase Plan/Execute Workflow Parity Implementation Plan

## Overview

This plan implements feature parity between the myles branch (Vercel AI SDK migration) and the main-testing-fixes branch for the two-phase plan/execute workflow. The fundamental change is modifying the myles branch to conditionally disable tools during plan generation, matching the main branch behavior where plans describe *what will be done* without actually executing tool calls.

## Current State Analysis

### Main Branch (Go) - Two-Phase Workflow

The main branch uses a clear separation between planning and execution:

**Phase 1: Plan Generation (NO TOOLS)**
- System prompt: `initialPlanSystemPrompt` or `updatePlanSystemPrompt`
- Instructions: `initialPlanInstructions` or `updatePlanInstructions` (include "don't write code")
- User message injection: `"Describe the plan only (do not write code) to create/edit a helm chart..."`
- LLM call: **No tools parameter** - forces descriptive text output
- Result: Detailed plan text describing what will be done

**Phase 2: Execution (WITH TOOLS)**
- System prompt: `executePlanSystemPrompt`
- Tools: `text_editor` (view, create, str_replace)
- Result: Sequential file creation/modification

Key code references:
- `pkg/llm/initial-plan.go:56` - "Describe the plan only (do not write code)" injection
- `pkg/llm/plan.go:69` - Same injection with dynamic verb (create/edit)
- `pkg/llm/initial-plan.go:60-64` - LLM call **without Tools parameter**
- `pkg/llm/create-knowledge.go:16-27` - `initialPlanInstructions` ("don't write code")

### Myles Branch (AI SDK) - Current Single-Phase

The myles branch currently uses a single-phase approach where tools are always enabled:

**Current Flow:**
1. User sends message
2. Intent classification via Groq (`isPlan=true`)
3. AI SDK `streamText()` called **with tools enabled**
4. Tool calls are buffered (not executed immediately)
5. `onFinish` creates plan from buffered tool calls
6. User sees plan (showing buffered tool calls)
7. "Proceed" button executes buffered tools

**Problem:** The AI SDK system prompt (`CHARTSMITH_TOOL_SYSTEM_PROMPT`) instructs the AI to "MUST use textEditor tool", causing immediate tool usage rather than descriptive planning.

Key code references:
- `lib/ai/prompts.ts:27-29` - "CRITICAL: When asked to create a file, you MUST use the textEditor tool"
- `app/api/chat/route.ts:207-241` - Tools always passed to `streamText()`
- `lib/ai/intent.ts:105-106` - `isPlan` routes to default "ai-sdk" handling (with tools)

## Desired End State

After implementation, the myles branch will match main branch behavior:

1. When `intent.isPlan=true`: Call AI SDK **without tools**, using plan-only system prompt
2. Plan response: Descriptive text explaining what will be done (no tool calls)
3. User reviews plan and clicks "Create Chart"/"Proceed"
4. When `intent.isProceed=true`: Call AI SDK **with tools**, using execution prompt
5. Tools execute sequentially, creating/modifying files

### Verification

- Test prompt: "create a simple nginx deployment"
- Expected Phase 1 output: Multi-paragraph plan describing files, architecture, features
- Expected Phase 2 output: 13+ files created via tool calls
- UI: "Proposed Plan (planning)" → "Create Chart" button → file-by-file creation

## What We're NOT Doing

- NOT implementing the intermediate `<chartsmithActionPlan>` XML generation step (main branch optional feature)
- NOT implementing bootstrap chart summary (can be added later)
- NOT changing the tool name (`textEditor` vs `text_editor_20241022`)
- NOT implementing embedding-based relevant file selection for update plans (can be added later)
- NOT changing UI button text (keeping "Proceed" - terminology change is cosmetic)
- NOT fixing UI showing 3-panel workspace immediately (see Future Work below)

## Implementation Approach

The implementation follows a minimal-change philosophy: modify the chat route to branch based on intent, using different prompts and tool configurations for each phase.

---

## Phase 1: Add Plan-Generation System Prompt

### Overview

Create a new system prompt specifically for plan generation that instructs the AI to describe plans without writing code. This mirrors the Go `initialPlanSystemPrompt` + `initialPlanInstructions`.

### Changes Required:

#### 1.1 Add CHARTSMITH_PLAN_SYSTEM_PROMPT

**File**: `chartsmith-app/lib/ai/prompts.ts`
**Changes**: Add new plan-generation prompt after `CHARTSMITH_CHAT_PROMPT`

```typescript
/**
 * Plan-generation system prompt - NO TOOL DIRECTIVES
 *
 * This prompt is used during Phase 1 (plan generation) when intent.isPlan=true.
 * It instructs the AI to describe the plan WITHOUT using tools or writing code.
 * Mirrors Go: initialPlanSystemPrompt + initialPlanInstructions
 */
export const CHARTSMITH_PLAN_SYSTEM_PROMPT = `You are ChartSmith, an expert AI assistant and a highly skilled senior software developer specializing in the creation, improvement, and maintenance of Helm charts.

Your primary responsibility is to help users transform, refine, and optimize Helm charts. Your guidance should be exhaustive, thorough, and precisely tailored to the user's needs.

## System Constraints

- Focus exclusively on Helm charts and Kubernetes manifests
- Do not assume external services unless explicitly mentioned

## Code Formatting

- Use 2 spaces for indentation in all YAML files
- Ensure YAML and Helm templates are valid and syntactically correct
- Use proper Helm templating expressions ({{ ... }}) where appropriate

## Response Format

- Use only valid Markdown for responses
- Do not use HTML elements
- Be concise and precise

## Planning Instructions

When asked to plan changes:
- Describe a general plan for creating or editing the helm chart based on the user request
- The user is a developer who understands Helm and Kubernetes
- You can be technical in your response, but don't write code
- Minimize the use of bullet lists in your response
- Be specific when describing the types of environments and versions of Kubernetes and Helm you will support
- Be specific when describing any dependencies you are including

<testing_info>
  - The user has access to an extensive set of tools to evaluate and test your output.
  - The user will provide multiple values.yaml to test the Helm chart generation.
  - For each change, the user will run \`helm template\` with all available values.yaml and confirm that it renders into valid YAML.
  - For each change, the user will run \`helm upgrade --install --dry-run\` with all available values.yaml and confirm that there are no errors.
</testing_info>

NEVER use the word "artifact" in your final messages to the user.`;

/**
 * Get plan-only user message injection
 * Mirrors Go: pkg/llm/initial-plan.go:56 and pkg/llm/plan.go:69
 */
export function getPlanOnlyUserMessage(isInitialPrompt: boolean): string {
  const verb = isInitialPrompt ? "create" : "edit";
  return `Describe the plan only (do not write code) to ${verb} a helm chart based on the previous discussion.`;
}
```

#### 1.2 Export new prompt

**File**: `chartsmith-app/lib/ai/prompts.ts`
**Changes**: Add exports for new prompt and function

```typescript
const prompts = {
  CHARTSMITH_TOOL_SYSTEM_PROMPT,
  CHARTSMITH_PLAN_SYSTEM_PROMPT,  // Add this
  CHARTSMITH_CHAT_PROMPT,
  CHARTSMITH_DEVELOPER_PROMPT,
  CHARTSMITH_OPERATOR_PROMPT,
  getSystemPromptWithContext,
  getSystemPromptForPersona,
  getPlanOnlyUserMessage,  // Add this
};
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles: `cd chartsmith-app && npm run build`
- [x] Linting passes: `cd chartsmith-app && npm run lint`
- [x] New exports are accessible: `import { CHARTSMITH_PLAN_SYSTEM_PROMPT, getPlanOnlyUserMessage } from '@/lib/ai/prompts'`

#### Manual Verification:
- [ ] Code review confirms prompt matches Go `initialPlanInstructions` semantics

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 2: Add Plan Intent Route Type

### Overview

Update the intent routing to explicitly handle plan generation as a distinct route type, rather than falling through to the default "ai-sdk" handler.

### Changes Required:

#### 2.1 Add "plan" route type

**File**: `chartsmith-app/lib/ai/intent.ts`
**Changes**: Add "plan" to IntentRoute union type

```typescript
/**
 * Intent routing result
 * Determines how to handle a message based on classified intent
 */
export type IntentRoute =
  | { type: "off-topic" }
  | { type: "proceed" }
  | { type: "render" }
  | { type: "plan" }      // NEW: Plan generation phase (no tools)
  | { type: "ai-sdk" };   // Default - let AI SDK handle with tools
```

#### 2.2 Update routeFromIntent to handle isPlan

**File**: `chartsmith-app/lib/ai/intent.ts`
**Changes**: Add plan routing logic before default case

```typescript
export function routeFromIntent(
  intent: Intent,
  isInitialPrompt: boolean,
  currentRevision: number
): IntentRoute {
  // IsProceed - execute the existing plan
  if (intent.isProceed) {
    return { type: "proceed" };
  }

  // IsOffTopic - polite decline (only if not initial message and has revision)
  if (intent.isOffTopic && !isInitialPrompt && currentRevision > 0) {
    return { type: "off-topic" };
  }

  // IsRender - trigger render
  if (intent.isRender) {
    return { type: "render" };
  }

  // IsPlan - generate plan description (NO TOOLS)
  // This is the key parity change: plan requests go to plan-only prompt
  if (intent.isPlan && !intent.isConversational) {
    return { type: "plan" };
  }

  // Default - let AI SDK process with tools (conversational, etc.)
  return { type: "ai-sdk" };
}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles: `cd chartsmith-app && npm run build`
- [ ] Unit tests pass (if any): `cd chartsmith-app && npm run test`

#### Manual Verification:
- [ ] Intent classification for "create a simple nginx deployment" returns `isPlan=true`
- [ ] Route returns `{ type: "plan" }` for plan intents

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 3: Modify Chat Route for Two-Phase Workflow

### Overview

This is the core change: modify the `/api/chat/route.ts` to branch based on intent route type. When `route.type === "plan"`, call AI SDK **without tools** using the plan-only prompt. When `route.type === "proceed"` or default, call AI SDK **with tools**.

### Changes Required:

#### 3.1 Import new prompts

**File**: `chartsmith-app/app/api/chat/route.ts`
**Changes**: Update imports to include plan prompt and helper

```typescript
import {
  getSystemPromptForPersona,
  CHARTSMITH_PLAN_SYSTEM_PROMPT,
  getPlanOnlyUserMessage,
} from '@/lib/ai/prompts';
```

#### 3.2 Add plan generation handler

**File**: `chartsmith-app/app/api/chat/route.ts`
**Changes**: Replace the switch statement in the intent handling section (around line 174) with expanded logic

```typescript
        switch (route.type) {
          case "off-topic":
            // Return polite decline without calling AI SDK
            console.log('[/api/chat] Declining off-topic message');
            return new Response(
              JSON.stringify({
                message: "I'm designed to help with Helm charts. Could you rephrase your question to be about Helm chart development or operations?"
              }),
              { status: 200, headers: { 'Content-Type': 'application/json' } }
            );

          case "plan": {
            // PLAN GENERATION PHASE: No tools, plan-only prompt
            // This matches Go behavior: pkg/llm/initial-plan.go:60-64 (no Tools param)
            console.log('[/api/chat] Plan generation phase - NO TOOLS');

            // Inject the "describe plan only" user message
            const planOnlyMessage = getPlanOnlyUserMessage(isInitialPrompt);
            const messagesWithPlanPrompt = [
              ...convertToModelMessages(messages),
              { role: 'user' as const, content: planOnlyMessage },
            ];

            const planResult = streamText({
              model: modelInstance,
              system: CHARTSMITH_PLAN_SYSTEM_PROMPT,
              messages: messagesWithPlanPrompt,
              // NO TOOLS - forces descriptive text response
              // This is the key difference from main branch parity
            });

            return planResult.toUIMessageStreamResponse();
          }

          case "proceed":
            // EXECUTION PHASE: Full tools enabled
            // Note: Full proceed handling will trigger proceedPlanAction
            // For now, let AI SDK handle it with tools (user saying "proceed" triggers execution)
            console.log('[/api/chat] Proceed intent detected, passing to AI SDK with tools');
            break;

          case "render":
            // Note: Render handling would typically trigger render pipeline
            // For now, let AI SDK acknowledge the render request
            console.log('[/api/chat] Render intent detected, passing to AI SDK');
            break;

          case "ai-sdk":
            // Continue to AI SDK processing below with tools
            break;
        }
```

#### 3.3 Refactor existing streamText call for clarity

**File**: `chartsmith-app/app/api/chat/route.ts`
**Changes**: Add comment explaining this is the execution/conversational path (with tools)

Around line 207 (the existing streamText call), add clarifying comment:

```typescript
    // EXECUTION/CONVERSATIONAL PATH: Tools enabled for file operations
    // This handles:
    // - route.type === "proceed" (execute existing plan)
    // - route.type === "ai-sdk" (conversational with potential tool use)
    // - route.type === "render" (acknowledge render request)
    //
    // Plan generation (route.type === "plan") is handled above and returns early.
    const result = streamText({
      model: modelInstance,
      system: systemPrompt,
      messages: convertToModelMessages(messages),
      tools, // PR1.5: Tools for chart operations
      stopWhen: stepCountIs(5),
      onFinish: async ({ finishReason, usage }) => {
        // ... existing onFinish logic
      },
    });
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles: `cd chartsmith-app && npm run build`
- [x] Linting passes: `cd chartsmith-app && npm run lint`
- [x] App starts without errors: `cd chartsmith-app && npm run dev`

#### Manual Verification:
- [ ] Test prompt "create a simple nginx deployment":
  - [ ] Response is descriptive plan text (multiple paragraphs)
  - [ ] Response does NOT contain tool calls
  - [ ] Response mentions file types to be created (Chart.yaml, values.yaml, templates/)
  - [ ] Response discusses Kubernetes version support, architecture decisions
- [ ] "Proceed" button appears after plan response
- [ ] Clicking "Proceed" triggers tool execution

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the two-phase workflow is functioning correctly before proceeding to the next phase.

---

## Phase 4: Verify Plan Creation and Storage

### Overview

Ensure that plans generated in the new two-phase flow are properly stored and associated with chat messages. The existing `createPlanFromToolCalls` function expects buffered tool calls, but in the plan phase we generate text only. We need to handle this case.

### Changes Required:

#### 4.1 Handle plan-only responses (no buffered tools)

**File**: `chartsmith-app/app/api/chat/route.ts`
**Changes**: The plan generation path already returns early and doesn't create a plan from tool calls. This is correct behavior - the plan text IS the plan, not buffered tool calls.

However, we should consider whether the frontend needs to know this is a "plan" response to show the appropriate UI. Currently the frontend shows plans based on the presence of a `workspace_plan` record. For the new flow:

**Option A (Simpler)**: Create plan record with `plan_text` field storing the AI response
**Option B (Current behavior)**: Frontend detects plan intent and shows plan UI for any response after a plan intent

For this implementation, we'll use **Option B** - the frontend already handles streaming responses. The plan UI will be triggered by the chat message's association with a plan intent.

**No code changes needed in this phase** - the existing flow handles this. The frontend already renders streaming AI responses and the `PlanChatMessage` component will display appropriately based on plan status.

#### 4.2 (Future Enhancement) Store plan text in workspace_plan

If needed later, we could modify the plan generation path to:
1. After streaming completes, extract the full response text
2. Create a `workspace_plan` record with `plan_text` column
3. Associate with the chat message

This would enable features like:
- Displaying the plan text in a dedicated "Proposed Plan" card
- Allowing users to edit the plan before execution

For now, this is out of scope.

### Success Criteria:

#### Automated Verification:
- [x] No code changes in this phase (verification only)
- [x] App continues to function after Phase 3 changes

#### Manual Verification:
- [ ] Plan response streams to UI correctly
- [ ] "Proceed" button functionality is preserved
- [ ] Clicking "Proceed" creates files as expected

**Implementation Note**: This phase is primarily verification. Proceed to Phase 5 after confirming the flow works end-to-end.

---

## Phase 5: Testing and Validation

### Overview

Comprehensive testing to ensure the two-phase workflow matches main branch behavior.

### Test Cases:

#### 5.1 Initial Plan Generation

**Test**: Create new workspace, send "create a simple nginx deployment"

**Expected**:
- Intent classification: `isPlan=true`
- Route: `{ type: "plan" }`
- Response: Detailed plan text (no tool calls)
- Content includes:
  - Target Kubernetes versions (1.24-1.31)
  - Helm version support (3.10+)
  - Files to be created (Chart.yaml, values.yaml, templates/)
  - Architecture decisions (Deployment, Service, Ingress, etc.)
  - Security considerations (RBAC, ServiceAccount)
  - Scaling features (HPA)

#### 5.2 Plan Execution (Proceed)

**Test**: After plan appears, click "Proceed"/"Create Chart"

**Expected**:
- Intent classification: `isProceed=true`
- Route: `{ type: "proceed" }`
- Tool calls executed sequentially
- 10+ files created
- UI shows file-by-file progress

#### 5.3 Update Plan Generation

**Test**: After initial chart created, send "add a configmap for nginx configuration"

**Expected**:
- Intent classification: `isPlan=true`
- Route: `{ type: "plan" }`
- Response: Plan describing configmap addition (no tool calls)
- Content references existing chart structure

#### 5.4 Conversational Messages

**Test**: Send "what files does a helm chart typically have?"

**Expected**:
- Intent classification: `isConversational=true`, `isPlan=false`
- Route: `{ type: "ai-sdk" }`
- Response: Informative answer (may or may not use tools)

#### 5.5 Off-Topic Messages (after chart exists)

**Test**: After chart created, send "what's the weather today?"

**Expected**:
- Intent classification: `isOffTopic=true`
- Route: `{ type: "off-topic" }`
- Response: Polite decline message

### Success Criteria:

#### Automated Verification:
- [x] All existing tests pass (build passes, linting passes)
- [x] No TypeScript errors
- [ ] No console errors in browser

#### Manual Verification:
- [ ] Test case 5.1 passes
- [ ] Test case 5.2 passes
- [ ] Test case 5.3 passes
- [ ] Test case 5.4 passes
- [ ] Test case 5.5 passes
- [ ] Comparison with main branch output shows similar plan quality

---

## Testing Strategy

### Unit Tests

- Intent routing: Verify `routeFromIntent` returns correct types
- Prompt selection: Verify correct prompt used for each route type

### Integration Tests

- End-to-end plan generation without tools
- End-to-end plan execution with tools
- Sequential file creation verification

### Manual Testing Steps

1. Start fresh workspace
2. Enter "create a simple nginx deployment"
3. Verify plan response is descriptive text (not tool calls)
4. Verify plan mentions specific files, versions, features
5. Click "Proceed"
6. Verify files are created sequentially
7. Verify 10+ files created matching main branch output
8. Send follow-up message "add resource limits"
9. Verify new plan is generated
10. Click "Proceed" again
11. Verify modifications applied correctly

## Performance Considerations

- Plan generation should be fast (no tool roundtrips)
- Execution phase will be slower (sequential tool calls)
- No regression in streaming performance expected

## Migration Notes

This is a non-breaking change:
- Existing workspaces continue to work
- No database migrations required
- No API changes required
- Frontend automatically adapts based on response type

## Future Work (Out of Scope)

### UI State Management: Single vs 3-Panel View

The main branch shows a single chat panel during plan generation, then transitions to the 3-panel workspace only after "Create Chart" is clicked. The myles branch currently shows the 3-panel workspace immediately.

**Root Cause**: UI routing doesn't distinguish between plan-pending vs plan-approved states.

**Potential Fix** (for future PR):
```typescript
// Track plan workflow state
type PlanState =
  | 'initial'      // No plan yet - single panel
  | 'planning'     // AI generating plan - single panel
  | 'review'       // User reviewing plan - single panel with "Create Chart" button
  | 'executing'    // Files being created - 3-panel workspace
  | 'complete';    // Done - 3-panel workspace with accept/reject

// Conditionally show file explorer
{(planState === 'executing' || planState === 'complete') && <FileExplorer />}
```

This is a separate UI concern that can be addressed after the core two-phase workflow is working correctly.

## References

- Research document: `/docs/testing/MYLES_BRANCH_PARITY_RESEARCH.md`
- Parity analysis: `/docs/testing/MYLES_BRANCH_PARITY_ANALYSIS.md`
- Main branch test flow: `/docs/testing/MAIN_BRANCH_NGINX_DEPLOYMENT_TEST.md`
- Main branch plan code: `pkg/llm/initial-plan.go`, `pkg/llm/plan.go`
- Main branch prompts: `pkg/llm/system.go`, `pkg/llm/create-knowledge.go`
