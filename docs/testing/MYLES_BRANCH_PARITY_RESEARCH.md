---
date: 2025-12-07T00:35:00Z
researcher: Claude Code
git_commit: 53bcaf8f98225f61c3ca37c572c0822b97ce23b3
branch: main
repository: UnchartedTerritory
topic: "Myles Branch vs Main-Testing-Fixes Feature Parity Analysis"
tags: [research, codebase, ai-sdk, plan-workflow, parity-analysis]
status: complete
last_updated: 2025-12-07
last_updated_by: Claude Code
---

# Research: Myles Branch vs Main-Testing-Fixes Feature Parity Analysis

**Date**: 2025-12-07T00:35:00Z
**Researcher**: Claude Code
**Git Commit**: 53bcaf8f98225f61c3ca37c572c0822b97ce23b3
**Branch**: main (research conducted across main-testing-fixes and myles/vercel-ai-sdk-migration)
**Repository**: UnchartedTerritory

## Research Question

Evaluate the accuracy of the MYLES_BRANCH_PARITY_ANALYSIS.md document and recommend changes needed on the myles branch to achieve feature parity with main-testing-fixes for the nginx deployment test flow.

## Executive Summary

The parity analysis document is **accurate**. The fundamental architectural difference is:

| Aspect | Main-Testing-Fixes (Go) | Myles Branch (AI SDK) |
|--------|------------------------|----------------------|
| **Workflow** | Two-phase: Plan → Execute | Single-phase with tool buffering |
| **Plan Generation** | Separate LLM call, NO tools | Same LLM call WITH tools |
| **User Approval** | "Create Chart" button | "Proceed" button |
| **Plan Content** | Descriptive text (what will be done) | Buffered tool calls (what was attempted) |
| **System Prompt** | "Describe the plan only (do not write code)" | "MUST use textEditor tool" |

## Detailed Findings

### 1. Main Branch (Go) Two-Phase Workflow

#### Phase 1: Plan Generation

**Entry Point**: `pkg/listener/new-plan.go:23`

When `intent.IsPlan=true`, the system creates a plan and routes to either:
- `createInitialPlan()` (when `CurrentRevision == 0`)
- `createUpdatePlan()` (when `CurrentRevision > 0`)

**Critical Pattern - Plan-Only System Prompt**:

The Go backend uses **two separate prompts** that explicitly prohibit code generation during planning:

1. **initialPlanInstructions** (`pkg/llm/create-knowledge.go:16-27`):
```
- Describe a general plan for creating a new helm chart based on the user request.
- You can be technical in your response, but don't write code.
```

2. **updatePlanInstructions** (`pkg/llm/create-knowledge.go:29-38`):
```
- Describe a general plan for editing an existing helm chart based on the user request.
- You can be technical in your response, but don't write code.
```

**Critical Pattern - User Message Injection**:

The final user message explicitly reinforces the plan-only behavior:

```go
// pkg/llm/initial-plan.go:56
initialUserMessage := "Describe the plan only (do not write code) to create a helm chart based on the previous discussion. "

// pkg/llm/plan.go:69
initialUserMessage := fmt.Sprintf("Describe the plan only (do not write code) to %s a helm chart based on the previous discussion. ", verb)
```

**Critical Pattern - No Tools During Plan Generation**:

The plan generation LLM calls do **NOT** include tools. Looking at `pkg/llm/initial-plan.go:60-64`:
```go
stream := client.Messages.NewStreaming(context.TODO(), anthropic.MessageNewParams{
    Model:     anthropic.F(anthropic.Model("claude-sonnet-4-20250514")),
    MaxTokens: anthropic.F(int64(8192)),
    Messages:  anthropic.F(messages),
    // NOTE: No Tools parameter!
})
```

#### Phase 2: Execution (After "Create Chart" Button)

When user clicks "Create Chart", the system:

1. Creates a revision (`pkg/workspace/revision.go:47`)
2. Enqueues `execute_plan` work (`pkg/workspace/revision.go:142-147`)
3. The `execute_plan` handler generates detailed action plans with XML tags
4. Enqueues `apply_plan` work
5. The `apply_plan` handler processes files sequentially using `text_editor` tool

**Execution System Prompt** (`pkg/llm/system.go:111-120`):
```
<execution_instructions>
  1. You will be asked to or edit a single file for a Helm chart.
  2. You will be given the current file. If it's empty, you should create the file to meet the requirements provided.
  ...
</execution_instructions>
```

### 2. Myles Branch (AI SDK) Current Implementation

#### Single-Phase with Tool Buffering

**Entry Point**: `chartsmith-app/app/api/chat/route.ts:59`

The current flow:
1. User sends message
2. Route calls `classifyIntent()` (same Groq classification as Go)
3. Route creates `bufferedToolCalls` array
4. AI SDK `streamText()` called with **tools enabled**
5. Tools intercept create/str_replace commands, buffer them
6. `onFinish` callback creates plan from buffered tools
7. Plan appears with "Proceed" button

**Critical Issue - Tool-Focused System Prompt**:

The myles branch prompts (`lib/ai/prompts.ts:27-29`) **directly contradict** plan-only behavior:

```typescript
**CRITICAL**: When asked to create a file, you MUST use the textEditor tool with command "create".
DO NOT just output the file contents in your response - actually create the file using the tool.

**CRITICAL**: When asked to modify a file, you MUST use the textEditor tool with command "str_replace".
DO NOT just show the changes - actually make them using the tool.
```

**Result**: The AI immediately uses tools instead of describing a plan.

### 3. Parity Analysis Validation

| Claim in MYLES_BRANCH_PARITY_ANALYSIS.md | Verification | Status |
|------------------------------------------|--------------|--------|
| Main branch uses two-phase plan/execute workflow | Confirmed via `pkg/listener/new-plan.go` and `pkg/listener/apply-plan.go` | ✅ ACCURATE |
| Main branch plan generation uses "Describe the plan only (do not write code)" | Confirmed at `pkg/llm/initial-plan.go:56` and `pkg/llm/plan.go:69` | ✅ ACCURATE |
| Myles branch goes straight to tool execution | Confirmed - prompts.ts lines 27-29 say "MUST use textEditor tool" | ✅ ACCURATE |
| Myles branch lacks plan-generation prompt | Confirmed - no equivalent to initialPlanInstructions | ✅ ACCURATE |
| Main branch uses `execute_plan` → `apply_plan` channels | Confirmed at `pkg/listener/start.go:47-61` | ✅ ACCURATE |

## Recommended Changes for Myles Branch

### Priority 1: Create Two-Phase Workflow

#### 1.1 Add New Plan-Generation System Prompt

Create a new prompt in `lib/ai/prompts.ts` that mirrors the Go behavior:

```typescript
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
  - For each change, the user will run helm template with all available values.yaml and confirm that it renders into valid YAML.
  - For each change, the user will run helm upgrade --install --dry-run with all available values.yaml and confirm that there are no errors.
</testing_info>

NEVER use the word "artifact" in your final messages to the user.`;
```

#### 1.2 Add User Message Injection for Plan Phase

In the route handler, when `intent.isPlan=true`, inject a final user message:

```typescript
// Add this message after the user's actual message
const planOnlyMessage = isInitialPrompt
  ? "Describe the plan only (do not write code) to create a helm chart based on the previous discussion."
  : "Describe the plan only (do not write code) to edit a helm chart based on the previous discussion.";
```

#### 1.3 Modify Route to Conditionally Disable Tools

```typescript
// In route.ts
if (intent.isPlan && !intent.isConversational) {
  // PLAN PHASE: No tools, plan-only prompt
  const result = streamText({
    model: modelInstance,
    system: CHARTSMITH_PLAN_SYSTEM_PROMPT,
    messages: [...convertToModelMessages(messages), { role: 'user', content: planOnlyMessage }],
    // NO TOOLS - force descriptive response
  });

  // Store plan description, set status to 'review'
  // ...
} else if (intent.isProceed) {
  // EXECUTION PHASE: Full tools, execution prompt
  const result = streamText({
    model: modelInstance,
    system: CHARTSMITH_TOOL_SYSTEM_PROMPT,
    messages: convertToModelMessages(messages),
    tools: createTools(...), // Full tool access
  });
}
```

### Priority 2: Implement Execution Phase

#### 2.1 Create Detailed Plan Generation (Optional but Recommended)

The main branch has an intermediate step that generates `<chartsmithActionPlan>` XML tags listing files to create/update. This is optional for parity but improves UX.

If implementing:
```typescript
export const CHARTSMITH_DETAILED_PLAN_PROMPT = `...
<planning_instructions>
  1. When asked to provide a detailed plan, expect that the user will provide a high level plan you must adhere to.
  2. Your final answer must be a \`<chartsmithArtifactPlan>\` block that completely describes the modifications needed:
     - Include a \`<chartsmithActionPlan>\` of type \`file\` for each file you expect to edit, create, or delete
     - Each \`<chartsmithActionPlan>\` must have a \`type\` attribute. Set this equal to \`file\`.
     - Each \`<chartsmithActionPlan>\` must have an \`action\` attribute. The valid actions are \`create\`, \`update\`, \`delete\`.
  3. Each \`<chartsmithActionPlan>\` must have a \`path\` attribute.
  4. Do not include any inner content in the \`<chartsmithActionPlan>\` tag.
</planning_instructions>`;
```

#### 2.2 Create Execution System Prompt

```typescript
export const CHARTSMITH_EXECUTION_PROMPT = `${CHARTSMITH_TOOL_SYSTEM_PROMPT}

<execution_instructions>
  1. You will be asked to create or edit a single file for a Helm chart.
  2. You will be given the current file. If it's empty, you should create the file to meet the requirements provided.
  3. If the file is not empty, you should update the file to meet the requirements provided.
  4. When editing an existing file, you should only edit the file to meet the requirements provided. Do not make any other changes to the file. Attempt to maintain as much of the current file as possible.
  5. You don't need to explain the change, just provide the artifact(s) in your response.
  6. Do not provide any other comments, just edit the files.
  7. Do not describe what you are going to do, just do it.
</execution_instructions>`;
```

### Priority 3: Update UI State Machine

The current myles branch UI already supports plan states (`review`, `applying`, `applied`). The changes needed:

1. **Plan Phase UI**: When `intent.isPlan=true`, display streaming text as plan description (not tool calls)
2. **"Create Chart" Button**: Add this button alongside/instead of "Proceed" to match main branch terminology
3. **Action Files Display**: When execution starts, display the list of files being processed

### Priority 4: Intent Routing Updates

Update `lib/ai/intent.ts` routing:

```typescript
export function routeFromIntent(
  intent: Intent,
  isInitialPrompt: boolean,
  currentRevision: number
): IntentRoute {
  // IsProceed - execute the plan
  if (intent.isProceed) {
    return { type: "execute" }; // New route type for execution phase
  }

  // IsPlan - generate plan description (NOT tool execution)
  if (intent.isPlan && !intent.isConversational) {
    return { type: "plan" }; // New route type for plan generation
  }

  // IsOffTopic
  if (intent.isOffTopic && !isInitialPrompt && currentRevision > 0) {
    return { type: "off-topic" };
  }

  // IsRender
  if (intent.isRender) {
    return { type: "render" };
  }

  // Default - conversational
  return { type: "conversational" };
}
```

## Implementation Sequence

1. **Phase A - Plan Generation Parity** (Critical)
   - Add `CHARTSMITH_PLAN_SYSTEM_PROMPT` to prompts.ts
   - Modify route.ts to disable tools when `intent.isPlan=true`
   - Inject "Describe the plan only" user message
   - Store plan description text (not buffered tool calls)

2. **Phase B - Execution Parity** (Required for full flow)
   - Keep existing `proceedPlanAction` for executing tools
   - Add execution prompt for when tools are enabled
   - Ensure sequential file processing

3. **Phase C - UI Parity** (Polish)
   - Rename "Proceed" to "Create Chart" for initial plans
   - Display action files list during execution
   - Match status indicators with main branch

## Architecture Diagram

```
Main Branch (Go) - Two Phase:
┌─────────────────────────────────────────────────────────────────────────┐
│  User: "create nginx deployment"                                        │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ Intent Classify │ → isPlan=true                                      │
│  └─────────────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ PHASE 1: Plan Generation                                    │        │
│  │ - System: initialPlanSystemPrompt + initialPlanInstructions │        │
│  │ - User: "Describe the plan only (do not write code)..."     │        │
│  │ - Tools: NONE                                               │        │
│  │ - Output: Descriptive plan text                             │        │
│  └─────────────────────────────────────────────────────────────┘        │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ UI: Plan Review │ → "Create Chart" button                            │
│  └─────────────────┘                                                    │
│           │                                                             │
│           ▼ (user clicks)                                               │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ PHASE 2: Execution                                          │        │
│  │ - execute_plan: Generate <chartsmithActionPlan> XML tags    │        │
│  │ - apply_plan: Sequential file processing                    │        │
│  │ - System: executePlanSystemPrompt                           │        │
│  │ - Tools: text_editor (view, str_replace, create)            │        │
│  └─────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘

Myles Branch (Current) - Single Phase:
┌─────────────────────────────────────────────────────────────────────────┐
│  User: "create nginx deployment"                                        │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ Intent Classify │ → isPlan=true (ignored, goes to AI SDK)            │
│  └─────────────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ SINGLE PHASE: AI SDK Streaming with Buffered Tools          │        │
│  │ - System: CHARTSMITH_TOOL_SYSTEM_PROMPT                     │        │
│  │   (says "MUST use textEditor tool")                         │        │
│  │ - Tools: ENABLED (but buffered)                             │        │
│  │ - Output: Tool calls buffered → Plan created in onFinish    │        │
│  └─────────────────────────────────────────────────────────────┘        │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ UI: Plan Review │ → "Proceed" button                                 │
│  │ (shows buffered │                                                    │
│  │  tool calls)    │                                                    │
│  └─────────────────┘                                                    │
│           │                                                             │
│           ▼ (user clicks)                                               │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ Execute buffered tool calls via proceedPlanAction           │        │
│  └─────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘

Myles Branch (Proposed) - Two Phase:
┌─────────────────────────────────────────────────────────────────────────┐
│  User: "create nginx deployment"                                        │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ Intent Classify │ → isPlan=true                                      │
│  └─────────────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ PHASE 1: Plan Generation (NEW)                              │        │
│  │ - System: CHARTSMITH_PLAN_SYSTEM_PROMPT (new)               │        │
│  │ - User: "Describe the plan only (do not write code)..."     │        │
│  │ - Tools: DISABLED                                           │        │
│  │ - Output: Descriptive plan text                             │        │
│  └─────────────────────────────────────────────────────────────┘        │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ UI: Plan Review │ → "Create Chart" button                            │
│  └─────────────────┘                                                    │
│           │                                                             │
│           ▼ (user clicks, sets isProceed=true)                          │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ PHASE 2: Execution                                          │        │
│  │ - System: CHARTSMITH_EXECUTION_PROMPT                       │        │
│  │ - Tools: ENABLED (textEditor)                               │        │
│  │ - Process files sequentially per action plan                │        │
│  └─────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
```

## Code References

### Main Branch (Go) - Key Files
- `pkg/llm/system.go:67-76` - initialPlanSystemPrompt
- `pkg/llm/system.go:78-87` - updatePlanSystemPrompt
- `pkg/llm/system.go:111-120` - executePlanSystemPrompt
- `pkg/llm/create-knowledge.go:16-27` - initialPlanInstructions
- `pkg/llm/create-knowledge.go:29-38` - updatePlanInstructions
- `pkg/llm/initial-plan.go:56` - "Describe the plan only" injection
- `pkg/llm/plan.go:69` - "Describe the plan only" injection (dynamic verb)
- `pkg/listener/new-plan.go:23` - Plan generation entry point
- `pkg/listener/execute-plan.go:24` - Detailed plan generation
- `pkg/listener/apply-plan.go:49` - Sequential file execution
- `pkg/llm/execute-action.go:437` - text_editor tool implementation

### Myles Branch - Key Files
- `chartsmith-app/lib/ai/prompts.ts:14-63` - CHARTSMITH_TOOL_SYSTEM_PROMPT
- `chartsmith-app/lib/ai/prompts.ts:27-29` - "MUST use textEditor" directives
- `chartsmith-app/app/api/chat/route.ts:59` - Chat route handler
- `chartsmith-app/lib/ai/tools/bufferedTools.ts` - Tool buffering logic
- `chartsmith-app/lib/workspace/actions/proceed-plan.ts:87` - Plan execution

## Open Questions

1. **Detailed Action Plan Step**: Should myles branch implement the intermediate `<chartsmithActionPlan>` XML generation step, or proceed directly from plan description to file execution?

2. **Bootstrap Chart**: Main branch includes a "bootstrap chart" summary in initial plan messages. Should this be replicated in myles branch?

3. **Relevant Files Selection**: Main branch uses embedding-based similarity (>=0.8, max 10 files) to choose relevant files for update plans. How should myles branch handle this for AI SDK?

4. **Tool Name**: Main branch uses `text_editor_20241022` (Sonnet 3.5 tool name). Myles branch uses `textEditor`. Should these be aligned?
