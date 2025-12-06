---
date: 2025-12-05T12:00:00-08:00
researcher: Claude
git_commit: 70f687ff55e3b646e62c776669da3129c8d64b3f
branch: main
repository: UnchartedTerritory
topic: "Missing Features Implementation Notes for Main Path Parity"
tags: [research, codebase, parity, ai-sdk-path, legacy-path, implementation-plan]
status: complete
last_updated: 2025-12-05
last_updated_by: Claude
---

# Research: Features Missing from Main Path (Post-PR2.0)

**Date**: 2025-12-05
**Git Commit**: 70f687ff55e3b646e62c776669da3129c8d64b3f
**Branch**: main
**Repository**: UnchartedTerritory

## Research Question

From PR1.9_FULL_PARITY_FEATURE_INVENTORY.md, these features were never implemented in the main AI SDK path:
1. Plan Display UI (PlanChatMessage)
2. Rollback functionality
3. Intent classification (Go)
4. K8s→Helm conversion
5. Followup action buttons
6. Page reload guard

Generate notes for building a plan to add these back to the main path.

---

## Summary

All six features exist in the legacy Go worker path with mature implementations. The main AI SDK path lacks these features entirely. Each feature has well-defined boundaries and can be implemented incrementally. The infrastructure (database schemas, type definitions, real-time events) largely exists - what's missing is the integration into the AI SDK chat flow.

---

## Feature 1: Plan Display UI (PlanChatMessage)

### What Exists in Legacy Path

| Component | Location | Purpose |
|-----------|----------|---------|
| `PlanChatMessage.tsx` | `components/PlanChatMessage.tsx` | Main plan display component |
| `plansAtom` | `atoms/workspace.ts:16` | Stores plans array |
| `planByIdAtom` | `atoms/workspace.ts:17-20` | Lookup function |
| `handlePlanUpdatedAtom` | `atoms/workspace.ts:132-147` | Real-time update handler |
| Plan status badges | `PlanChatMessage.tsx:192-200` | Color-coded status display |
| Action files section | `PlanChatMessage.tsx:233-294` | Expandable file list |
| Proceed/Ignore buttons | `PlanChatMessage.tsx:296-327` | User approval controls |

### Plan Status Lifecycle
```
pending → planning → review → applying → applied
                  ↘ ignored
```

### Database Schema
- Table: `workspace_plan` (id, description, status, workspace_id, chat_message_ids, created_at, proceed_at)
- Table: `workspace_plan_action_file` (plan_id, action, path, status)
- Message field: `response_plan_id` links messages to plans

### What's Missing in AI SDK Path
- No plan creation from AI SDK responses
- No plan display in chat messages
- No proceed/ignore workflow
- Plans atom is hydrated but never updated

### Implementation Notes

**Option A: Port Plan Workflow to AI SDK**
1. Add plan detection to AI SDK response processing
2. When AI proposes file changes, create a plan record first
3. Display `PlanChatMessage` instead of directly writing files
4. On "Proceed", execute file changes via existing tools
5. On "Ignore", mark plan as ignored and skip changes

**Option B: Use AI SDK Tools for Plan Management**
1. Create `proposePlan` tool that AI calls instead of `textEditor`
2. Tool creates plan record and returns plan ID
3. Frontend detects `responsePlanId` and shows `PlanChatMessage`
4. Proceed triggers `executePlan` tool call

**Key Files to Modify:**
- `app/api/chat/route.ts` - Add plan tool or post-processing
- `hooks/useAISDKChatAdapter.ts` - Handle plan responses
- `components/ChatMessage.tsx` - Render PlanChatMessage when `responsePlanId` exists
- `lib/workspace/actions/` - Add plan actions for AI SDK

**Complexity**: High (8-12 hours)
**Dependencies**: None

---

## Feature 2: Rollback Functionality

### What Exists in Legacy Path

| Component | Location | Purpose |
|-----------|----------|---------|
| `RollbackModal.tsx` | `components/RollbackModal.tsx` | 5-second countdown modal |
| `rollbackWorkspaceAction` | `lib/workspace/actions/rollback.ts:7` | Server action |
| `rollbackToRevision` | `lib/workspace/workspace.ts:871-942` | Database transaction |
| Rollback link | `ChatMessage.tsx:288-307` | Trigger in chat messages |

### Rollback Database Operations
1. Update `workspace.current_revision_number`
2. Delete `workspace_revision` rows for future revisions
3. Delete `workspace_rendered`, `workspace_rendered_chart`, `workspace_rendered_file` for future revisions
4. Delete `workspace_chat` messages for future revisions
5. Delete `workspace_plan`, `workspace_plan_action_file` for deleted messages
6. Delete `workspace_file`, `workspace_chart` for future revisions
7. Enqueue re-render for rolled-back revision

### `responseRollbackToRevisionNumber` Field
- Set on first message of each revision
- Used to show "rollback to this revision" link
- Only shown if: has value AND not current revision AND is first message for that revision

### What's Missing in AI SDK Path
- No rollback link in chat messages
- `RollbackModal` component exists but not triggered
- `responseRollbackToRevisionNumber` not set on AI SDK messages

### Implementation Notes

1. **Set revision marker on messages**: When AI SDK commits changes (creates new revision), set `responseRollbackToRevisionNumber` on the first message of that revision
2. **Add rollback link to ChatMessage**: Already exists at lines 288-307, just needs the field populated
3. **Wire up RollbackModal**: Already imported and functional

**Key Files to Modify:**
- `lib/workspace/actions/commit-pending-changes.ts` - Set `responseRollbackToRevisionNumber` on first message
- `hooks/useAISDKChatAdapter.ts` - Track revision boundaries

**Complexity**: Medium (4-6 hours)
**Dependencies**: Commit flow must track which message initiated the revision

---

## Feature 3: Intent Classification

### What Exists in Legacy Path

| Component | Location | Purpose |
|-----------|----------|---------|
| `intent.go` | `pkg/llm/intent.go:15-143` | Groq-based classification |
| `new_intent.go` | `pkg/listener/new_intent.go:24-349` | Intent handler & routing |
| Intent type | `pkg/workspace/types/types.go:117-125` | 7 boolean flags |
| `UpdateChatMessageIntent` | `pkg/workspace/intent.go:11-25` | Database persistence |

### Intent Types (7 total, not 8)
```go
type Intent struct {
    IsOffTopic       bool
    IsPlan           bool
    IsConversational bool
    IsChartDeveloper bool
    IsChartOperator  bool
    IsProceed        bool
    IsRender         bool
}
```
Note: `IsRollback` mentioned in PRD does NOT exist in code.

### Intent Routing Logic
| Intent | Handler | Action |
|--------|---------|--------|
| `IsProceed` | Creates revision | `workspace.CreateRevision()` |
| `IsOffTopic` | Declines politely | Streams decline response |
| `IsRender` | Triggers render | `workspace.EnqueueRenderWorkspace()` |
| `IsPlan` | Creates plan | `workspace.CreatePlan()` |
| `IsConversational` | Chat handler | Enqueues to `new_converational` |

### What's Missing in AI SDK Path
- All messages go to same LLM path
- No Groq-based pre-classification
- No smart routing based on intent
- Off-topic handling relies on LLM judgment only

### Implementation Notes

**Option A: Port Go Intent Detection (Exact Parity)**
- Add Groq API call before AI SDK
- Route based on intent
- Pros: Proven logic, exact parity
- Cons: Added latency (~300-500ms), complexity

**Option B: Build Intent Detection into AI SDK System Prompt**
- Modify system prompt to include intent detection instructions
- Use tool calls to signal intent (e.g., `renderChart` tool, `declineOffTopic` tool)
- Pros: Single LLM call, faster
- Cons: Less deterministic

**Option C: Skip Intent Detection**
- Let AI SDK handle all routing internally
- Pros: Simplest architecture
- Cons: No explicit off-topic handling, less control

**Recommendation**: Start with Option B for MVP. The AI SDK already has tools for rendering (`renderWorkspace`). Add `declineOffTopic` tool and let AI model decide.

**Key Files to Modify (Option B):**
- `app/api/chat/route.ts` - Add intent-related tools
- `lib/ai/system-prompts.ts` - Add intent detection instructions
- `app/api/chat/tools/` - Add `declineOffTopic` tool

**Complexity**: Very High (12-16 hours)
**Dependencies**: Requires architectural decision first

---

## Feature 4: K8s→Helm Conversion Pipeline

### What Exists in Legacy Path

| Component | Location | Purpose |
|-----------|----------|---------|
| `ConversionProgress.tsx` | `components/ConversionProgress.tsx:110` | 6-step progress UI |
| `FileList.tsx` | `components/FileList.tsx:18` | Per-file progress |
| `new-conversion.go` | `pkg/listener/new-conversion.go:23` | Pipeline init |
| `new-conversion-file.go` | `pkg/listener/new-conversion-file.go:23` | Per-file conversion |
| `conversion-normalize-values.go` | `pkg/listener/conversion-normalize-values.go:23` | Values cleanup |
| `conversion-simplify.go` | `pkg/listener/conversion-simplify.go:18` | Final assembly |

### 6-Step Pipeline
```
Analyzing → Sorting → Templating → Normalizing → Simplifying → Complete
```

### LLM Usage
- **Groq** (`llama-3.3-70b-versatile`): Per-file K8s→Helm conversion
- **Claude** (`claude-sonnet-4`): Values.yaml cleanup/normalization

### Database Schema
- Table: `workspace_conversion` (id, workspace_id, status, chart_yaml, values_yaml, ...)
- Table: `workspace_conversion_file` (id, conversion_id, file_path, file_content, converted_files, status)

### What's Missing in AI SDK Path
- Entire conversion feature missing
- No way to import K8s manifests
- `conversionsAtom` hydrated but never used
- `ConversionProgress` component exists but not integrated

### Implementation Notes

**This is a major feature requiring significant work.**

**Phase 1: Backend Integration**
1. Create conversion initiation endpoint/action
2. Port Go worker logic to TypeScript or create HTTP bridge
3. Use existing queue system for background processing

**Phase 2: Frontend Integration**
1. Add "Import K8s Manifests" flow
2. Display `ConversionProgress` during conversion
3. Handle real-time status updates via Centrifugo

**Key Decision**: Reuse Go workers or port to TypeScript?
- Go workers are battle-tested and use Groq efficiently
- Could create HTTP endpoint to trigger Go conversion from AI SDK path
- Alternatively, port entire pipeline to TypeScript

**Key Files:**
- Need new: `lib/workspace/actions/start-conversion.ts`
- Need new: `app/api/conversion/route.ts` or bridge to Go
- Integrate: `components/ConversionProgress.tsx`

**Complexity**: Very High (24-32 hours)
**Dependencies**: Architectural decision on Go vs TypeScript

---

## Feature 5: Followup Action Buttons

### What Exists in Legacy Path

| Component | Location | Purpose |
|-----------|----------|---------|
| `FollowupAction` type | `lib/types/workspace.ts:141-144` | Action/label pair |
| Button rendering | `ChatMessage.tsx:309-330` | UI buttons |
| `performFollowupAction` | `lib/workspace/actions/perform-followup-action.ts:7` | Click handler |
| Database column | `workspace_chat.followup_actions` | JSONB storage |

### Data Structure
```typescript
interface FollowupAction {
  action: string;  // e.g., "render"
  label: string;   // e.g., "Render the chart"
}
```

### Current Implementation
- Buttons appear below assistant messages
- Click calls `performFollowupAction(action)`
- Only "render" action is implemented
- Creates new message with known intent

### What's Missing in AI SDK Path
- `messageMapper.ts:162` explicitly notes "not supported in AI SDK mode"
- No followup actions generated
- UI code exists but never receives data

### Implementation Notes

**Option A: AI SDK Tool for Followup Actions**
1. Add `suggestFollowup` tool that AI can call
2. Tool stores followup actions on current message
3. UI renders buttons when message has followupActions

**Option B: Post-Response Analysis**
1. After AI SDK response completes, analyze content
2. If response mentions rendering, add "Render" followup
3. Store followup actions during persistence step

**Key Files to Modify:**
- `lib/chat/messageMapper.ts:162` - Add followupActions mapping
- `hooks/useAISDKChatAdapter.ts` - Generate followups after response
- `app/api/chat/tools/` - Add `suggestFollowup` tool (Option A)

**Complexity**: Medium (3-4 hours)
**Dependencies**: Message persistence must support followupActions field

---

## Feature 6: Page Reload Guard

### What Exists in Legacy Path

**Nothing.** Page reload guard does NOT exist in either path.

### What Infrastructure Exists

| Component | Location | Purpose |
|-----------|----------|---------|
| `content_pending` column | `workspace_file` table | Tracks uncommitted changes |
| `allFilesWithContentPendingAtom` | `atoms/workspace.ts:117-129` | Derived atom for dirty files |
| Commit action | `commit-pending-changes.ts` | Promotes pending to committed |
| Discard action | `discard-pending-changes.ts` | Clears pending changes |

### What's Missing (Both Paths)
- No `beforeunload` event handler
- No navigation guards
- No warning when leaving with uncommitted work
- No warning during streaming

### Implementation Notes

**Simple Implementation:**
```typescript
// In WorkspaceContent.tsx or ChatContainer.tsx
useEffect(() => {
  const handleBeforeUnload = (e: BeforeUnloadEvent) => {
    const hasPending = allFilesWithContentPending.length > 0;
    const isStreaming = chatState.status === 'streaming';

    if (hasPending || isStreaming) {
      e.preventDefault();
      e.returnValue = 'You have unsaved changes.';
      return 'You have unsaved changes.';
    }
  };

  window.addEventListener('beforeunload', handleBeforeUnload);
  return () => window.removeEventListener('beforeunload', handleBeforeUnload);
}, [allFilesWithContentPending, chatState.status]);
```

**Key Files to Modify:**
- `components/WorkspaceContent.tsx` or `components/ChatContainer.tsx` - Add useEffect

**Complexity**: Low (1-2 hours)
**Dependencies**: None (infrastructure exists)

---

## Implementation Plan (Based on Final Decisions)

### Phase 1: Quick Wins (3-4 hours)
| Feature | Approach | Hours |
|---------|----------|-------|
| Page Reload Guard | Add `beforeunload` handler using existing `allFilesWithContentPendingAtom` | 1-2 |
| Followup Actions | Wire existing UI (`ChatMessage.tsx:309-330`) to message persistence | 2-3 |

### Phase 2: Go Backend Integration (8-12 hours)
| Feature | Approach | Hours |
|---------|----------|-------|
| Intent Classification | Call Go's Groq endpoint before AI SDK, route based on intent | 4-6 |
| Rollback | Set `responseRollbackToRevisionNumber` on commit, wire existing `RollbackModal` | 4-6 |

### Phase 3: Plan Workflow (10-14 hours)
| Feature | Approach | Hours |
|---------|----------|-------|
| Tool Call Interception | Intercept `textEditor` calls, buffer instead of execute | 4-6 |
| Plan Creation | Call Go backend to create plan from buffered tool calls | 2-4 |
| Plan Display UI | Wire `PlanChatMessage` to AI SDK path when `responsePlanId` exists | 4-6 |

### Phase 4: K8s Conversion Bridge (4-8 hours)
| Feature | Approach | Hours |
|---------|----------|-------|
| HTTP Bridge | TypeScript endpoint triggering Go conversion pipeline | 2-4 |
| Frontend Integration | Wire `ConversionProgress.tsx` to AI SDK path | 2-4 |

**Total Estimated: 25-38 hours**

### Implementation Flow Diagrams

#### Intent Classification Flow
```
User Message
    → TypeScript calls Go /api/intent
    → Groq classifies (7 intents)
    → Route:
        - IsProceed → Go creates revision
        - IsRender → Go enqueues render
        - IsOffTopic → Go streams decline
        - IsPlan/IsConversational → AI SDK processes
```

#### Plan Workflow Flow
```
AI SDK receives message
    → AI calls `textEditor` tool (unchanged behavior)
    → TypeScript intercepts tool call, buffers it
    → Calls Go backend to create plan from buffered changes
    → Go returns plan ID, sets `responsePlanId` on message
    → Frontend detects `responsePlanId`
    → Shows PlanChatMessage with Proceed/Ignore
    → User clicks Proceed
    → Buffered tool calls are executed
    → Files written to `content_pending`
```

#### K8s Conversion Flow
```
User uploads K8s manifests
    → TypeScript calls Go /api/conversion
    → Go runs 6-step pipeline (existing code)
    → Centrifugo sends status events
    → ConversionProgress.tsx updates (existing component)
    → Conversion complete → new chart created
```

---

## Architectural Decisions (FINAL)

### Decision 1: Plan Workflow Approach
- **SELECTED: Option A - Post-Process AI Responses**
- AI uses existing `textEditor` tool (unchanged behavior)
- TypeScript/Go intercepts tool calls and creates plan record
- Frontend shows `PlanChatMessage` with Proceed/Ignore buttons
- On Proceed, intercepted tool calls are executed
- Rationale: Matches legacy pattern exactly - backend decides when to create plan, not AI. Preserves existing system prompts and AI behavior per original requirements.

### Decision 2: Intent Classification Approach
- **SELECTED: Option A - Port Go Groq Detection**
- Add Groq API call before AI SDK processes message
- Classify intent first, then route to appropriate handler
- Exact parity with legacy behavior
- Rationale: Strict parity required, 500ms latency acceptable, proven logic

### Decision 3: K8s Conversion Architecture
- **SELECTED: Option B - HTTP Bridge to Existing Go Workers**
- Create thin TypeScript endpoint that triggers Go conversion pipeline
- Reuse battle-tested Go code for 6-step conversion
- Centrifugo events update frontend (already wired)
- Rationale: Fastest path to parity, Go backend staying, tight timeline

---

## Code References

### Plans
- `components/PlanChatMessage.tsx` - Full component
- `atoms/workspace.ts:16-20, 132-147` - State management
- `lib/workspace/workspace.ts:422-469` - Plan creation
- `lib/types/workspace.ts:101-116` - Type definitions

### Rollback
- `components/RollbackModal.tsx` - Full component
- `lib/workspace/actions/rollback.ts:7-21` - Server action
- `lib/workspace/workspace.ts:871-942` - Database transaction
- `components/ChatMessage.tsx:288-307` - UI trigger

### Intent
- `pkg/llm/intent.go:15-143` - Classification logic
- `pkg/listener/new_intent.go:24-349` - Handler & routing
- `pkg/workspace/types/types.go:117-125` - Intent struct

### Conversion
- `components/ConversionProgress.tsx:110-289` - UI component
- `pkg/listener/new-conversion.go` - Pipeline init
- `pkg/listener/new-conversion-file.go` - Per-file processing
- `lib/types/workspace.ts:65-99` - Frontend types

### Followup Actions
- `lib/types/workspace.ts:141-144` - Type definition
- `components/ChatMessage.tsx:309-330` - UI rendering
- `lib/workspace/actions/perform-followup-action.ts:7-27` - Action handler

### Page Reload Guard
- `atoms/workspace.ts:117-129` - Dirty state tracking
- `lib/workspace/actions/commit-pending-changes.ts` - Commit action
- `lib/workspace/actions/discard-pending-changes.ts` - Discard action

---

## Open Questions

1. Should plan approval be required before any file changes, or only for multi-file changes?
2. Should intent classification affect which AI model is used (cheaper model for conversational)?
3. Is K8s conversion a priority, or can it be deferred to focus on core parity?
4. Should followup actions be AI-generated or rule-based (e.g., always show "Render" after file changes)?
