# PR1.9+ Full Feature Parity Inventory

**Status**: Research Complete  
**Purpose**: Comprehensive inventory of ALL features needed for 100% parity between `/test-ai-chat` and `/workspace/[id]`  
**Total Features Identified**: 28 (beyond PR1.8's 5)

---

## Overview

This document consolidates findings from two independent research analyses. After PR1.8, achieving full feature parity requires implementing **28 additional features** across approximately 6-10 PRs.

### PR1.8 Features (Already Planned)
1. Diff Visualization
2. Plan Persistence to Database
3. Render Tab Updates
4. Conflict Resolution
5. Page Reload Guard

---

## P0: Critical / Blocking (4 features)

### 1. Full Plan Display UI
**Main Path**: `PlanChatMessage.tsx`  
**What It Does**: Displays AI-proposed plans with status badges, action file lists, expand/collapse, proceed/ignore buttons  
**Current State**: AI SDK path has no plan display - changes go directly to files  
**Complexity**: High  
**Est. Hours**: 8-12

### 2. Plan Proceed/Ignore Actions
**Main Path**: `createRevisionAction`, `ignorePlanAction`  
**What It Does**: User control over whether to apply AI-proposed changes. Proceed creates a new revision; Ignore marks plan as superseded  
**Current State**: AI SDK commits directly without review step  
**Complexity**: Medium  
**Est. Hours**: 4-6

### 3. Intent Classification & Smart Routing
**Main Path**: `pkg/llm/intent.go`, `pkg/listener/new_intent.go`  
**What It Does**: Groq-based classification into 8 intents (IsConversational, IsOffTopic, IsPlan, IsChartDeveloper, IsChartOperator, IsProceed, IsRender, IsRollback). Routes to different handlers automatically.  
**Current State**: AI SDK sends all messages to same LLM path  
**Complexity**: Very High  
**Est. Hours**: 12-16

### 4. K8s Manifest to Helm Conversion Pipeline
**Main Path**: `ConversionProgress.tsx`, `pkg/listener/new-conversion.go`, `conversion-simplify.go`, `conversion-normalize-values.go`  
**What It Does**: 6-step pipeline converting Kubernetes manifests to Helm charts (Analyze → Sort → Template → Normalize → Simplify → Assemble)  
**Current State**: Entire feature missing from AI SDK path  
**Complexity**: Very High  
**Est. Hours**: 24-32

---

## P1: High Value (10 features)

### 5. Role/Persona Selector
**Main Path**: `ChatContainer.tsx` role dropdown  
**What It Does**: Dropdown with Auto/Developer/Operator options. Changes AI response perspective based on who's asking.  
**Current State**: No persona selection in AI SDK path  
**Complexity**: Low  
**Est. Hours**: 2-3

### 6. Plan Status Tracking
**Main Path**: `plansAtom`, `handlePlanUpdatedAtom`  
**What It Does**: Full lifecycle: `pending` → `planning` → `review` → `applying` → `applied` → `ignored`  
**Current State**: No plan state machine in AI SDK path  
**Complexity**: Medium  
**Est. Hours**: 4-6

### 7. Rollback Functionality
**Main Path**: `RollbackModal.tsx`, `responseRollbackToRevisionNumber`  
**What It Does**: UI to rollback workspace to any previous revision  
**Current State**: No rollback capability in AI SDK path  
**Complexity**: Medium  
**Est. Hours**: 4-6

### 8. Per-file Accept/Reject Patch Controls
**Main Path**: `CodeEditor.tsx` with Accept/Reject buttons  
**What It Does**: Accept or reject changes per-file (not just bulk commit/discard). Includes "Accept All" / "Reject All" options.  
**Current State**: AI SDK only has bulk Commit/Discard  
**Complexity**: Medium  
**Est. Hours**: 6-8

### 9. Auto-Render After Plan Apply
**Main Path**: `apply-plan.go` → `EnqueueRenderWorkspaceForRevisionWithPendingContent`  
**What It Does**: Automatically triggers Helm template render after changes are committed  
**Current State**: AI SDK requires manual render trigger  
**Complexity**: Low  
**Est. Hours**: 2-3

### 10. Message Field Completeness
**Main Path**: Message type with 15+ fields  
**What It Does**: Full message state tracking: `isIntentComplete`, `isApplied`, `isApplying`, `isIgnored`, `isCanceled`, `revisionNumber`, `responseRenderId`, `responsePlanId`, `responseConversionId`, `followupActions`, etc.  
**Current State**: AI SDK only persists basic `prompt`/`response`  
**Complexity**: Medium  
**Est. Hours**: 4-6

### 11. Comprehensive Initial Data Load
**Main Path**: `WorkspaceContent.tsx` loads 5 parallel fetches  
**What It Does**: Initial page load fetches: workspace, messages, plans, renders, conversions  
**Current State**: AI SDK only loads workspace + messages  
**Complexity**: Low  
**Est. Hours**: 2-3

### 12. Plans Atom Hydration
**Main Path**: `plansAtom` hydrated on mount  
**What It Does**: Plans state available throughout app via Jotai  
**Current State**: No plans atom usage in AI SDK path  
**Complexity**: Low  
**Est. Hours**: 1-2

### 13. Conversions Atom Hydration
**Main Path**: `conversionsAtom` hydrated on mount  
**What It Does**: Conversion state available throughout app via Jotai  
**Current State**: No conversions atom usage in AI SDK path  
**Complexity**: Low  
**Est. Hours**: 1-2

### 14. Adaptive Layout (New Chart Mode)
**Main Path**: `WorkspaceContent.tsx` layout switching  
**What It Does**: Different layouts for new chart creation (centered) vs. editing existing chart (3-panel)  
**Current State**: AI SDK has fixed 3-panel layout always  
**Complexity**: Medium  
**Est. Hours**: 3-4

---

## P2: Medium Value (8 features)

### 15. Terminal Component for Render Output
**Main Path**: `Terminal.tsx`  
**What It Does**: Displays live streaming output from `helm template` and `helm dep update` commands with collapsible UI  
**Current State**: No terminal visualization in AI SDK path  
**Complexity**: Medium  
**Est. Hours**: 4-6

### 16. Nested Chat Input for Plan Questions
**Main Path**: Chat input inside `PlanChatMessage.tsx`  
**What It Does**: Ability to ask questions/request changes while reviewing a plan  
**Current State**: No plan-specific chat input  
**Complexity**: Low  
**Est. Hours**: 2-3

### 17. Followup Action Buttons
**Main Path**: `followupActions` array in messages  
**What It Does**: Dynamic buttons for suggested next actions (e.g., "Render chart", "Add values file")  
**Current State**: No followup actions in AI SDK path  
**Complexity**: Medium  
**Est. Hours**: 3-4

### 18. Message Cancellation Persistence
**Main Path**: `cancelMessageAction`, `isCanceled` field  
**What It Does**: Cancel button during "thinking" state, persists cancellation to database  
**Current State**: AI SDK has `stop()` but no persistence  
**Complexity**: Low  
**Est. Hours**: 2-3

### 19. Off-Topic Detection & Handling
**Main Path**: `IsOffTopic` intent, `DeclineOffTopicChatMessage`  
**What It Does**: Graceful handling of non-Helm questions with appropriate responses  
**Current State**: Handled by LLM judgment only (no structured detection)  
**Complexity**: Medium  
**Est. Hours**: 3-4

### 20. Command Menu (⌘+K)
**Main Path**: `CommandMenuWrapper.tsx`, `CommandMenu.tsx`  
**What It Does**: Quick actions palette triggered by ⌘+K  
**Current State**: No command menu in AI SDK path  
**Complexity**: Medium  
**Est. Hours**: 4-6

### 21. Rendered Files List Display
**Main Path**: In `Terminal.tsx` with checkmarks  
**What It Does**: Shows list of successfully rendered files with status  
**Current State**: Not visible in AI SDK path  
**Complexity**: Low  
**Est. Hours**: 1-2

### 22. Conversion Progress Display
**Main Path**: `ConversionProgress.tsx`  
**What It Does**: 6-step progress UI for K8s to Helm conversion  
**Current State**: Component not integrated (requires #4 first)  
**Complexity**: Medium (if #4 done)  
**Est. Hours**: 4-6

---

## P3: Lower Value (6 features)

### 23. Debug Panel
**Main Path**: `DebugPanel.tsx`  
**What It Does**: Developer debugging information display  
**Current State**: Not in AI SDK path  
**Complexity**: Low  
**Est. Hours**: 2-3

### 24. File Delete Functionality
**Main Path**: `DeleteFileModal.tsx`  
**What It Does**: Confirmation modal for file deletion, integration with file tree  
**Current State**: No file deletion in AI SDK path (textEditor only has view/create/str_replace)  
**Complexity**: Low  
**Est. Hours**: 2-3

### 25. Feedback Modal
**Main Path**: `FeedbackModal.tsx`  
**What It Does**: Thumbs up/down feedback collection, submission to backend  
**Current State**: No feedback UI in AI SDK path  
**Complexity**: Low  
**Est. Hours**: 2-3

### 26. Toast Notifications
**Main Path**: `toast/*.tsx` components  
**What It Does**: Standardized toast notification system  
**Current State**: No toast system in AI SDK path  
**Complexity**: Low  
**Est. Hours**: 1-2

### 27. New Chart Content Component
**Main Path**: `NewChartContent.tsx`, `NewChartChatMessage.tsx`  
**What It Does**: Special UI for initial chart creation flow  
**Current State**: Basic landing page exists but not full parity  
**Complexity**: Medium  
**Est. Hours**: 3-4

### 28. Side Navigation Parity
**Main Path**: `SideNav.tsx`, `SideNavWrapper.tsx`  
**What It Does**: Full sidebar with all navigation options  
**Current State**: AI SDK has minimal sidebar  
**Complexity**: Low  
**Est. Hours**: 2-3

---

## Summary Statistics

| Priority | Count | Est. Hours |
|----------|-------|------------|
| P0 Critical | 4 | 48-66 |
| P1 High | 10 | 30-45 |
| P2 Medium | 8 | 23-34 |
| P3 Low | 6 | 12-18 |
| **Total** | **28** | **113-163** |

---

## Suggested PR Roadmap

```
PR1.8: (Already Planned - ~6-10 hours)
  └── Diff Visualization, Plan Persistence, Render Tab, Conflict, Reload Guard

PR1.9: Plan Display Foundation (~12-16 hours)
  └── Features #1, #6 - Plan UI + Status Tracking

PR1.10: Plan Actions (~8-12 hours)  
  └── Features #2, #16 - Proceed/Ignore + Nested Chat

PR1.11: Enhanced Patch Controls (~8-12 hours)
  └── Feature #8 - Per-file Accept/Reject

PR1.12: Data & State Completeness (~10-14 hours)
  └── Features #10, #11, #12, #13 - Message Fields + Atom Hydration

PR1.13: UX Polish (~12-16 hours)
  └── Features #5, #14, #17, #18 - Role Selector, Layout, Followups, Cancellation

PR1.14: Rollback & Terminal (~10-14 hours)
  └── Features #7, #15, #21 - Rollback, Terminal, Rendered Files

PR1.15: Intent Classification (~14-18 hours)
  └── Features #3, #19 - Smart Routing + Off-Topic

PR1.16: K8s Conversion (Major) (~28-38 hours)
  └── Features #4, #22 - Full Conversion Pipeline

PR1.17: Polish & Remaining (~14-20 hours)
  └── Features #9, #20, #23-28 - Command Menu, Debug, Delete, Feedback, etc.
```

---

## Architecture Decision Required

Before implementing P0 features, a key decision is needed:

**Option A: Port Go Intent Detection to AI SDK Path**
- Pros: Exact parity, proven logic
- Cons: Added latency (Groq call before main LLM), complexity

**Option B: Build Intent Detection into AI SDK System Prompt**
- Pros: Single LLM call, faster
- Cons: Less deterministic, may need tuning

**Option C: Skip Intent Detection Entirely**
- Pros: Simpler architecture
- Cons: No smart routing, all messages processed same way

**Recommendation**: Start with Option B for MVP, optionally add Option A later for exact parity.

---

## Files to Study

| Feature Area | Key Files |
|--------------|-----------|
| Plan Workflow | `PlanChatMessage.tsx`, `plansAtom`, `pkg/listener/new-plan.go`, `execute-plan.go`, `apply-plan.go` |
| Intent | `pkg/llm/intent.go`, `pkg/listener/new_intent.go` |
| Conversion | `ConversionProgress.tsx`, `pkg/listener/new-conversion*.go` |
| Rollback | `RollbackModal.tsx`, rollback actions |
| Terminal | `Terminal.tsx`, `render-workspace.go` |
| Patch Controls | `CodeEditor.tsx`, `accept-patch.ts`, `reject-patch.ts` |

---

*This inventory provides the complete scope for achieving 100% feature parity. Implementation order should prioritize P0/P1 features that block core workflows.*

*Last Updated: Dec 5, 2025*

