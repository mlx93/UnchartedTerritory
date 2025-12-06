---
date: 2025-12-06T21:54:22Z
researcher: mlx93
git_commit: bc6b607c349e5519d1677b1f3defb4982b92cbba
branch: myles/vercel-ai-sdk-migration
repository: chartsmith
topic: "Feature Parity Analysis: Main Branch vs AI SDK Migration Branch"
tags: [research, codebase, ai-sdk, feature-parity, helm, kubernetes, conversion]
status: complete
last_updated: 2025-12-06
last_updated_by: mlx93
---

# Research: Feature Parity Analysis - Main Branch vs AI SDK Migration Branch

**Date**: 2025-12-06T21:54:22Z
**Researcher**: mlx93
**Git Commit**: bc6b607c349e5519d1677b1f3defb4982b92cbba
**Branch**: myles/vercel-ai-sdk-migration
**Repository**: chartsmith

## Research Question

Verify feature parity between `origin/main` and `myles/vercel-ai-sdk-migration` branches, specifically checking:
1. K8s → Helm conversion end-to-end with AI SDK path
2. textEditor tool delete command
3. Command Menu (⌘+K) in AI SDK mode
4. Rollback functionality with AI SDK-created revisions
5. Auto-render after plan functionality
6. Feedback Modal integration
7. ConversionProgress UI real-time updates

## Summary

| Feature | Main Branch | Myles Branch | Status | Gap? |
|---------|-------------|--------------|--------|------|
| K8s → Helm Conversion | Full 6-step pipeline | Identical implementation | ✅ Parity | No |
| ConversionProgress UI | N/A (feature added in myles) | Full implementation with real-time | ✅ New Feature | No |
| textEditor Delete | Not implemented | Not implemented | ⚠️ Missing | **Yes - Both** |
| Command Menu (⌘+K) | Full implementation | Identical, mode-agnostic | ✅ Parity | No |
| Rollback Functionality | Full implementation | Identical + AI SDK support | ⚠️ Partial | **NewChartChatMessage bug** |
| Auto-Render After Plan | Automatic in apply-plan.go | **Missing** in proceedPlanAction | ❌ Gap | **Yes - Myles Only** |
| Feedback Modal | UI exists, no backend | Identical - UI only | ⚠️ Incomplete | Both (design issue) |

## Detailed Findings

---

### 1. K8s → Helm Conversion Pipeline

**Status: ✅ FULL PARITY**

The conversion pipeline is **identical** between both branches with the same 6-step workflow:

1. **Analyzing** - Scan K8s resources
2. **Sorting** - Map dependencies by GVK priority
3. **Templating** - Convert each file via LLM (Groq llama-3.3-70b)
4. **Normalizing** - Clean up values.yaml via Claude
5. **Simplifying** - Final assembly
6. **Complete** - Package Helm chart

**Key Files (Identical in Both):**
- `chartsmith-app/lib/ai/tools/convertK8s.ts:20-82` - AI SDK tool entry point
- `chartsmith-app/lib/ai/conversion.ts:47-64` - `startConversion()` function
- `pkg/api/handlers/conversion.go:40-95` - Go HTTP handler
- `pkg/listener/new-conversion.go:23-129` - Pipeline stage 1
- `pkg/listener/new-conversion-file.go:23-157` - Pipeline stage 2 (per-file)
- `pkg/listener/conversion-normalize-values.go:23-87` - Pipeline stage 3
- `pkg/listener/conversion-simplify.go:18-78` - Pipeline stage 4
- `pkg/llm/convert-file.go:24-351` - LLM integration

**Data Flow:**
```
AI Tool → POST /api/conversion/start → PostgreSQL Queue → Workers → Centrifugo Events → UI
```

---

### 2. ConversionProgress UI Real-Time Updates

**Status: ✅ NEW FEATURE (Myles Branch Only)**

ConversionProgress.tsx is a **new component** added in the myles branch - it does not exist in origin/main.

**Component Location:** `chartsmith-app/components/ConversionProgress.tsx`

**6 UI Steps Displayed (lines 119-156):**
1. "Analyzing Manifests" - Scanning Kubernetes resources
2. "Sorting to find dependencies" - Mapping relationships
3. "Creating initial templates" - Converting manifests to Helm
4. "Normalizing values" - Identifying parameters
5. "Simplifying templates" - Optimizing configurations
6. "Final assembly" - Packaging complete chart

**Real-Time Update Mechanism:**
- Uses **Centrifugo** WebSocket events
- Two event types: `conversion-status` and `conversion-file`
- Frontend subscribes via `useCentrifugo.ts:526-644`
- State managed via Jotai atoms (`conversionsAtom`, `handleConversionUpdatedAtom`)
- `useEffect` at lines 188-224 syncs step statuses

**File-Level Progress:** `chartsmith-app/components/FileList.tsx`
- Shows during Templating and Simplifying steps
- Windowed view of up to 30 files
- Auto-scrolls to active file
- Status icons: pending (gray), converting (blue spinner), converted (green check)

---

### 3. textEditor Tool Delete Command

**Status: ❌ MISSING IN BOTH BRANCHES**

The textEditor tool supports only 3 commands:
- `view` - Read file contents
- `create` - Create new file
- `str_replace` - Replace text in file

**No delete command exists.**

**Evidence:**
- `chartsmith-app/lib/ai/tools/textEditor.ts:49` - Enum: `['view', 'create', 'str_replace']`
- `pkg/api/handlers/editor.go:108-117` - Switch only handles view/create/str_replace
- Default case returns: "Invalid command. Supported: view, create, str_replace"

**UI Components Exist But Non-Functional:**
- `chartsmith-app/components/DeleteFileModal.tsx` - Full modal UI exists
- `chartsmith-app/components/FileTree.tsx:203-215` - Delete button and handler exist
- **But:** `FileTree.tsx:485-490` - `onConfirm` handler is **empty** - closes modal without deleting

**Required Files Protected:**
- Chart.yaml and values.yaml marked as `isRequired` and cannot be deleted

---

### 4. Command Menu (⌘+K)

**Status: ✅ FULL PARITY - Mode Agnostic**

The Command Menu works identically in both AI SDK mode and legacy mode.

**Key Files:**
- `chartsmith-app/components/CommandMenu.tsx` - Main UI component
- `chartsmith-app/components/CommandMenuWrapper.tsx` - Dynamic loading wrapper
- `chartsmith-app/contexts/CommandMenuContext.tsx` - Context with keyboard registration

**Keyboard Trigger:** `mod+k` (⌘+K on macOS, Ctrl+K on Windows/Linux)
- Registered at `CommandMenuContext.tsx:20-31` via `react-hotkeys-hook`
- Also registered in Monaco editors at `CodeEditor.tsx:126-137`

**Available Commands:**
1. **Theme switching** - Light/Dark/Auto (lines 77-94)
2. **Debug Terminal toggle** - Show/Hide debug panel (lines 95-109)

**Why Mode-Agnostic:**
- Command menu reads from Jotai atoms populated by either chat transport
- No conditional logic based on `NEXT_PUBLIC_USE_AI_SDK_CHAT`
- Context provider wraps entire app before chat mode selection

---

### 5. Rollback Functionality

**Status: ⚠️ PARTIAL GAP - Bug in NewChartChatMessage**

The core rollback logic is **identical** between branches and works with AI SDK-created revisions.

**Core Implementation (Identical):**
- `chartsmith-app/components/RollbackModal.tsx` - 5-second countdown modal
- `chartsmith-app/lib/workspace/actions/rollback.ts` - Action handler
- `chartsmith-app/lib/workspace/workspace.ts:874-945` - Transactional rollback

**Rollback Process:**
1. Updates `current_revision_number` to target
2. Deletes all future revisions (cascade: renders, files, plans, chats)
3. Re-queues render job for rolled-back revision

**AI SDK Revision Creation:**
- `chartsmith-app/lib/workspace/actions/commit-pending-changes.ts:31-96`
- Creates revision with `created_type = 'ai_sdk_commit'`
- Sets `response_rollback_to_revision_number` on first message of previous revision

**BUG FOUND - NewChartChatMessage.tsx:**
- Shows rollback button at lines 271-286
- **Missing RollbackModal import** (unlike ChatMessage.tsx line 14)
- **No RollbackModal component instantiation** (unlike ChatMessage.tsx lines 388-400)
- Button click calls `setShowRollbackModal(true)` but modal never renders
- **Result:** Clicking rollback button does nothing in NewChartChatMessage

**ChatMessage.tsx Works Correctly:**
- Import at line 14: `import { RollbackModal } from "@/components/RollbackModal"`
- Modal rendered at lines 388-400

---

### 6. Auto-Render After Plan

**Status: ❌ GAP - Missing in Myles Branch**

**Origin/Main Branch (Works):**
- Auto-render triggered in `pkg/listener/apply-plan.go:160-170`
- After plan application completes, calls `EnqueueRenderWorkspaceForRevisionWithPendingContent()`
- Creates render record with `is_autorender: true`
- UI filters auto-renders unless they fail

**Myles Branch (Missing):**
- `proceedPlanAction` at `chartsmith-app/lib/workspace/actions/proceed-plan.ts:61-196`
- Executes buffered tool calls
- Updates plan status to 'applied'
- **NO call to trigger auto-render** - function ends at line 196

**Flow Comparison:**

| Step | Main Branch | Myles Branch |
|------|------------|--------------|
| Plan execution | Go worker via `apply_plan` queue | TypeScript `proceedPlanAction` |
| Tool execution | LLM regenerates content | Buffered tool calls executed directly |
| Auto-render | ✅ `EnqueueRenderWorkspaceForRevision` | ❌ Not triggered |
| Render type | Uses `content_pending` | N/A |

**Additional Gap:**
- `chartsmith-app/lib/workspace/workspace.ts:40-41` skips auto-render for `ChatMessageIntent.NON_PLAN` (AI SDK conversational intent)

---

### 7. Feedback Modal

**Status: ⚠️ INCOMPLETE IN BOTH BRANCHES (Design Issue)**

The FeedbackModal is a **presentation-only component** with no backend integration.

**Component:** `chartsmith-app/components/FeedbackModal.tsx`

**UI Features:**
- 5-star rating system (lines 110-134)
- Text feedback textarea (lines 137-146)
- Context display: prompt, files sent, response

**Integration Points:**
- `ChatMessage.tsx:379-386` - Modal rendered but **no trigger button**
- `NewChartChatMessage.tsx:353-360` - Same issue
- `PlanChatMessage.tsx:322-342` - **Only working integration:**
  - ThumbsUp button opens modal
  - ThumbsDown button calls `handleIgnore()` directly (skips modal)

**Critical Issue - No Data Persistence:**
```typescript
// FeedbackModal.tsx:42-47
const handleSubmit = (e: React.FormEvent) => {
  e.preventDefault();
  if (description.trim()) {
    onClose();  // Just closes modal - NO API CALL
  }
};
```

**Missing:**
- No API endpoint for feedback submission
- No database table for storing feedback
- No analytics tracking
- Feedback is collected and immediately discarded

---

## Architecture Documentation

### AI SDK Chat Integration

The myles branch introduces a dual-mode chat system controlled by feature flag:

**Feature Flag:** `NEXT_PUBLIC_USE_AI_SDK_CHAT`
- `true` (default): AI SDK mode via `/api/chat`
- `false`: Legacy Go worker mode

**AI SDK Path:**
1. Chat route at `app/api/chat/route.ts`
2. Tool calls buffered via `createBufferedTools()`
3. Plan created in `onFinish` callback via `createPlanFromToolCalls()`
4. User proceeds → `proceedPlanAction()` executes buffered tools

**Legacy Path:**
1. Intent classification
2. Go worker generates plan via LLM
3. User proceeds → `createRevisionAction()` → `execute_plan` queue
4. Go worker applies plan → auto-render triggered

### Real-Time Updates

Both branches use **Centrifugo** for real-time WebSocket events:
- Workspace changes
- Conversion progress
- Plan status updates
- Render progress

---

## Code References

### K8s Conversion
- `chartsmith-app/lib/ai/tools/convertK8s.ts:20-82` - AI tool definition
- `pkg/api/handlers/conversion.go:40-95` - HTTP handler
- `pkg/listener/new-conversion.go:23-129` - Stage 1 processor

### textEditor Tool
- `chartsmith-app/lib/ai/tools/textEditor.ts:34-89` - Tool definition
- `pkg/api/handlers/editor.go:108-117` - Command switch
- `chartsmith-app/components/DeleteFileModal.tsx` - UI component (non-functional)

### Command Menu
- `chartsmith-app/contexts/CommandMenuContext.tsx:20-31` - Keyboard registration
- `chartsmith-app/components/CommandMenu.tsx:54-114` - UI component

### Rollback
- `chartsmith-app/components/RollbackModal.tsx` - Modal component
- `chartsmith-app/lib/workspace/workspace.ts:874-945` - Core rollback logic
- `chartsmith-app/components/NewChartChatMessage.tsx:271-286` - **BUG: Missing modal**

### Auto-Render
- `pkg/listener/apply-plan.go:160-170` - Main branch trigger (works)
- `chartsmith-app/lib/workspace/actions/proceed-plan.ts:61-196` - Myles branch (missing)

### Feedback Modal
- `chartsmith-app/components/FeedbackModal.tsx:42-47` - Empty submit handler
- `chartsmith-app/components/PlanChatMessage.tsx:322-342` - Only working trigger

### ConversionProgress
- `chartsmith-app/components/ConversionProgress.tsx:119-156` - Step definitions
- `chartsmith-app/hooks/useCentrifugo.ts:466-474` - Event handlers

---

## Gaps Requiring Action

### 1. Auto-Render After Plan (Critical)
**Location:** `chartsmith-app/lib/workspace/actions/proceed-plan.ts`
**Issue:** No render triggered after AI SDK plan execution
**Fix:** Add call to render workspace after updating plan status to 'applied'

### 2. RollbackModal in NewChartChatMessage (Bug)
**Location:** `chartsmith-app/components/NewChartChatMessage.tsx`
**Issue:** Missing RollbackModal import and component
**Fix:** Add import and render RollbackModal like ChatMessage.tsx

### 3. textEditor Delete Command (Enhancement)
**Location:** Both branches
**Issue:** Delete operation not supported in textEditor tool
**Fix:**
- Add 'delete' to command enum in `textEditor.ts`
- Add delete handler in `pkg/api/handlers/editor.go`
- Connect FileTree.tsx onConfirm to call delete endpoint

### 4. Feedback Modal Backend (Enhancement)
**Location:** Both branches
**Issue:** Feedback collected but not persisted
**Fix:**
- Create feedback API endpoint
- Create database table
- Update FeedbackModal handleSubmit to call API

---

## Open Questions

1. Should auto-render be optional for AI SDK path (user preference)?
2. What files should be deletable via textEditor tool? (currently protected: Chart.yaml, values.yaml)
3. Where should feedback be stored - local database or external analytics service?
4. Should the ConversionProgress component be backported to main branch?

---

## Related Research

- `/Users/mylessjs/Desktop/UnchartedTerritory/PRDs/` - PR documentation
- `/Users/mylessjs/Desktop/UnchartedTerritory/Replicated_Chartsmith.md` - Requirements document
