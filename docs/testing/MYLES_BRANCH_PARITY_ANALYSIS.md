# Myles Branch Parity Analysis: Critical Differences from Main

**Date:** 2025-12-06  
**Branch:** `myles/vercel-ai-sdk-migration`  
**Test Prompt:** `create a simple nginx deployment`  
**Workspace URL:** http://localhost:3000/workspace/0hgY22mLgPLE  
**Status:** ⚠️ SIGNIFICANT PARITY GAPS IDENTIFIED

---

## Executive Summary

The myles branch has **fundamental architectural differences** from main that result in a completely different user experience. The root cause is a **missing two-phase plan workflow** - the AI SDK implementation goes straight to execution instead of generating a plan for user review first.

---

## Side-by-Side Comparison

| Aspect | Main Branch | Myles Branch | Issue |
|--------|-------------|--------------|-------|
| **Initial View** | Single chat panel, no file explorer | 3-panel workspace with file explorer | ❌ Major |
| **AI Response Style** | Detailed plan text (multiple paragraphs) | Stream-of-consciousness execution narration | ❌ Major |
| **Plan Presentation** | "Proposed Plan (planning)" card with markdown | No plan card, just response text | ❌ Major |
| **Action Button** | "Create Chart" | "Proceed" | ⚠️ Minor |
| **When Execution Happens** | AFTER user clicks "Create Chart" | DURING initial response (buffered) | ❌ Major |
| **Files Created** | 13 files | 2 files | ❌ Major |
| **File Changes Display** | Progressive, one-by-one as they're created | All at once after "Proceed" | ❌ Major |
| **User Control** | Full control - review plan, then approve | Execution already decided, just revealing results | ❌ Major |

---

## Root Cause Analysis

### The Fundamental Problem: Missing Two-Phase Workflow

**Main Branch (Go) has TWO phases:**

```
Phase 1: PLAN GENERATION
├── Uses initialPlanSystemPrompt
├── User message: "Describe the plan only (do not write code)"
├── AI generates DETAILED PLAN TEXT (no tool calls)
├── User reviews plan
└── User clicks "Create Chart"

Phase 2: PLAN EXECUTION
├── Uses executePlanSystemPrompt  
├── AI uses text_editor tool to make changes
├── Files processed sequentially
└── Real-time updates to UI
```

**Myles Branch (AI SDK) has ONE phase:**

```
Single Phase: IMMEDIATE EXECUTION
├── Uses CHARTSMITH_TOOL_SYSTEM_PROMPT
├── Prompt says "MUST use textEditor tool" immediately
├── AI goes straight to tool calls
├── Tool calls are buffered, but AI never generates plan text
└── "Proceed" button reveals already-decided changes
```

### System Prompt Comparison

**Main Branch (`pkg/llm/initial-plan.go` line 56):**
```go
initialUserMessage := "Describe the plan only (do not write code) to create a helm chart based on the previous discussion."
```

**Myles Branch (`lib/ai/prompts.ts` lines 27-29):**
```typescript
**CRITICAL**: When asked to create a file, you MUST use the textEditor tool with command "create". 
Do NOT just output the file contents in your response - actually create the file using the tool.
```

The main branch **explicitly tells the AI NOT to write code** during planning.
The myles branch **explicitly tells the AI TO use tools immediately**.

---

## Detailed Issue Breakdown

### Issue 1: Immediate Workspace View

**Expected (Main):**
- User lands on "Create a new Helm chart" page
- Single chat panel, clean interface
- File explorer only appears AFTER plan approval

**Actual (Myles):**
- User immediately sees 3-panel workspace
- File explorer visible from the start
- Suggests execution mode, not planning mode

**Root Cause:** UI routing doesn't distinguish between plan-pending vs plan-approved states.

---

### Issue 2: No Plan Text Generated

**Expected (Main):**
```markdown
## Plan for Creating a Simple NGINX Deployment Helm Chart

I will create a production-ready Helm chart that deploys NGINX as a web server 
with comprehensive configuration options and enterprise-grade features...

**Target Environment Support**
The chart will support Kubernetes versions 1.24 through 1.31...

**Core Application Architecture**
The chart will deploy nginx as a stateless web application using a Deployment...
```

**Actual (Myles):**
```
I'll create a simple Helm chart for an nginx deployment. Let me start by checking 
the current chart context and then create the necessary files. I can see there's 
already a chart structure in place. I need to create the deployment template to 
complete the nginx deployment. Let me create the deployment file: Now let me 
update the Chart.yaml to reflect that this is an nginx chart: Let me also remove 
the replicated dependency since this is a simple nginx chart
```

**Root Cause:** System prompt drives immediate tool usage. AI narrates actions instead of describing a plan.

---

### Issue 3: Execution Already Done Before User Approval

**Expected (Main):**
1. AI generates plan text
2. User reviews plan
3. User clicks "Create Chart"
4. THEN execution begins
5. Files created one-by-one with real-time updates

**Actual (Myles):**
1. AI immediately decides to execute
2. Tool calls are buffered (but decision is made)
3. AI narrates what it did/is doing
4. "Proceed" button appears
5. Clicking reveals already-buffered changes all at once

**Root Cause:** Tool buffering happens AFTER the AI decides to execute, not before. The AI's decision to use tools is immediate due to the system prompt.

---

### Issue 4: Only 2 Files vs 13 Files

**Main Branch creates 13 files:**
- Chart.yaml, values.yaml, _helpers.tpl
- deployment.yaml, service.yaml, serviceaccount.yaml
- ingress.yaml, hpa.yaml, configmap.yaml
- pdb.yaml, networkpolicy.yaml, servicemonitor.yaml, NOTES.txt

**Myles Branch creates 2 files:**
- templates/deployment.yaml
- Chart.yaml

**Root Cause:** Different prompting leads to different AI behavior. Main branch's detailed planning prompt encourages comprehensive chart creation. Myles branch's immediate execution prompt leads to minimal changes.

---

## Required Changes for Parity

### Change 1: Implement Two-Phase Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                    PHASE 1: PLAN GENERATION                     │
├─────────────────────────────────────────────────────────────────┤
│  1. User sends prompt                                           │
│  2. Intent classification: is_plan=true                         │
│  3. Call AI SDK with PLAN-GENERATION prompt (no tools)          │
│  4. AI generates detailed plan text                             │
│  5. Store plan text, show "Proposed Plan" card                  │
│  6. User clicks "Create Chart"                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PHASE 2: PLAN EXECUTION                      │
├─────────────────────────────────────────────────────────────────┤
│  1. "Create Chart" triggers execution                           │
│  2. Call AI SDK with EXECUTION prompt + tools                   │
│  3. Tool calls buffered and executed sequentially               │
│  4. Real-time updates to UI as each file completes              │
│  5. Accept/Reject buttons appear when done                      │
└─────────────────────────────────────────────────────────────────┘
```

### Change 2: Create Plan-Generation System Prompt

Need a new prompt that:
- Tells AI to describe the plan WITHOUT using tools
- Matches main branch's planning style
- Generates comprehensive, detailed plan text

```typescript
export const CHARTSMITH_PLAN_PROMPT = `You are ChartSmith...

When asked to create or modify a Helm chart, you must FIRST describe your plan 
in detail WITHOUT making any changes. Do NOT use any tools. Do NOT write code.

Your plan should describe:
- Target environment support (Kubernetes versions, distributions)
- Core architecture decisions
- Files you will create or modify
- Configuration and customization options
- Security considerations
- Operational features

After the user approves your plan, they will click a button to proceed with 
the actual implementation.`;
```

### Change 3: Route Plan Requests to Plan-Generation Phase

```typescript
// In /api/chat route
if (intent.isPlan && !intent.isProceed) {
  // Use plan-generation prompt, NO tools
  return streamText({
    model: modelInstance,
    system: CHARTSMITH_PLAN_PROMPT,  // Plan prompt, not tool prompt
    messages: convertToModelMessages(messages),
    // NO tools parameter - plan phase doesn't use tools
  });
}

// Only after "Proceed/Create Chart"
if (intent.isProceed) {
  // Use execution prompt WITH tools
  return streamText({
    model: modelInstance,
    system: CHARTSMITH_EXECUTION_PROMPT,
    tools: createBufferedTools(...),  // Tools only in execution phase
    ...
  });
}
```

### Change 4: UI State Management

```typescript
// Track plan workflow state
type PlanState = 
  | 'initial'      // No plan yet
  | 'planning'     // AI generating plan
  | 'review'       // User reviewing plan
  | 'executing'    // Files being created
  | 'complete';    // Done, accept/reject

// Show file explorer only in executing/complete states
{planState === 'executing' || planState === 'complete' && <FileExplorer />}
```

---

## Implementation Priority

| Priority | Change | Effort | Impact |
|----------|--------|--------|--------|
| P0 | Two-phase workflow routing | High | Critical |
| P0 | Plan-generation system prompt | Medium | Critical |
| P1 | UI state management for plan phases | Medium | High |
| P1 | "Create Chart" button instead of "Proceed" | Low | Medium |
| P2 | Progressive file change display | Medium | Medium |

---

## Files to Modify

| File | Change |
|------|--------|
| `lib/ai/prompts.ts` | Add CHARTSMITH_PLAN_PROMPT |
| `app/api/chat/route.ts` | Route plan vs execution phases |
| `lib/ai/intent.ts` | Ensure plan detection works correctly |
| `components/ChatMessage.tsx` | Handle plan vs execution rendering |
| `components/PlanChatMessage.tsx` | May need updates for new flow |
| `hooks/useAISDKChatAdapter.ts` | Track plan workflow state |

---

## Summary

The myles branch's AI SDK integration **skips the planning phase entirely**. The system prompt drives immediate tool usage, which conflicts with the main branch's two-phase approach (plan first, execute after approval).

**Key insight:** Tool buffering was implemented correctly, but it captures tool calls AFTER the AI decides to execute. The fix requires routing plan requests to a tools-free planning phase first, then only enabling tools after user approval.

---

*Created: Dec 6, 2025*
*Related: docs/testing/MAIN_BRANCH_NGINX_DEPLOYMENT_TEST.md*

