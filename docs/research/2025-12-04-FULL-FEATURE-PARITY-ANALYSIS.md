---
date: 2025-12-04T22:18:23-06:00
researcher: mylessjs
git_commit: 70f687ff55e3b646e62c776669da3129c8d64b3f
branch: main
repository: mlx93/UnchartedTerritory
topic: "Full Feature Parity Analysis: /workspace/[id] vs /test-ai-chat/[workspaceId]"
tags: [research, feature-parity, vercel-ai-sdk, chartsmith, migration]
status: complete
last_updated: 2025-12-04
last_updated_by: mylessjs
---

# Research: Full Feature Parity Analysis Between Main Workspace and AI SDK Test Path

**Date**: 2025-12-04T22:18:23-06:00
**Researcher**: mylessjs
**Git Commit**: 70f687ff55e3b646e62c776669da3129c8d64b3f
**Branch**: main
**Repository**: mlx93/UnchartedTerritory

## Research Question

What additional features outside of PR1.8_FULL_PARITY_ROADMAP.md are missing for feature parity between `/workspace/[id]` (main path) and `/test-ai-chat/[workspaceId]` (AI SDK path)?

---

## Summary

After comprehensive analysis of both paths, **22 additional feature gaps** were identified beyond the 5 features listed in PR1.8. The main workspace path is a mature, full-featured implementation with Go backend integration, while the test-ai-chat path is a functional prototype focused on direct AI SDK streaming with file editing capabilities.

### PR1.8 Already Documents (5 Features):
1. Diff Visualization
2. Plan Persistence to Database
3. Render Tab Updates
4. Conflict Resolution
5. Page Reload Guard

### Additional Gaps NOT in PR1.8 (22 Features):
See detailed breakdown below.

---

## Detailed Findings

### Category 1: Chat & AI Interaction Features

#### 1.1 Role/Persona Selector (Missing)
**Main Path**: `ChatContainer.tsx:140-201`
- Dropdown with three options: Auto-detect, Chart Developer, End User
- Each role uses different icons (Sparkles, Code, User)
- Affects how AI interprets and responds to messages
- Stored as `messageFromPersona` in chat message

**Test Path**: No persona selection - always uses conversational mode

**Priority**: P2 (Affects AI response quality)

---

#### 1.2 Message Cancel Button During Generation (Partial)
**Main Path**: `ChatMessage.tsx:242-254`
- Shows "Cancel" button during `thinking` state
- Calls cancel action to abort Go worker processing
- Shows "canceled" message after cancellation

**Test Path**: Has `stop()` function from useChat but UI may differ
- Location: `client.tsx:569-582`

**Priority**: P3 (Polish)

---

#### 1.3 Followup Action Buttons (Missing)
**Main Path**: `ChatMessage.tsx:309-329`
- Shows action buttons after AI response
- Enables quick followup interactions
- Appears below completed messages

**Test Path**: No followup action buttons

**Priority**: P3 (Polish)

---

#### 1.4 Rollback Functionality (Missing)
**Main Path**:
- `ChatMessage.tsx:288-307` - Rollback link on first message of revision
- `RollbackModal.tsx` - 5-second countdown with progress bar
- `rollbackWorkspaceAction` - Server action to revert to previous revision

**Test Path**: No rollback UI or functionality

**Priority**: P2 (Data recovery feature)

---

### Category 2: Plan Workflow Features

#### 2.1 Full Plan Display UI (Missing)
**Main Path**: `PlanChatMessage.tsx:38-393`
- Displays plan description with markdown
- Shows status badge (planning/pending/review/applying/applied/ignored)
- Collapsible list of action files with icons (Plus/Pencil/Trash)
- Status animations for file operations
- Proceed/Ignore action buttons

**Test Path**: No plan UI - only uses `planProposal` tool internally

**Priority**: P1 (Core feature for plan-based workflows)

---

#### 2.2 Plan Proceed/Ignore Actions (Missing)
**Main Path**: `PlanChatMessage.tsx:296-327`
- ThumbsUp/ThumbsDown buttons for feedback
- "Proceed" button creates revision and applies plan
- "Ignore" button marks plan as superseded

**Test Path**: No plan actions - changes applied directly via tools

**Priority**: P1 (User control over AI changes)

---

#### 2.3 Nested Chat Input for Plan Questions (Missing)
**Main Path**: `PlanChatMessage.tsx:328-375`
- Inline chat input below plan to ask questions
- Allows iterating on plan before proceeding

**Test Path**: No nested chat - single chat stream only

**Priority**: P2 (Plan iteration UX)

---

#### 2.4 Plan Status Tracking (Missing)
**Main Path**: Tracks plan through states:
- `pending` → `planning` → `review` → `applying` → `applied`
- Or `ignored` if user rejects
- Real-time status updates via Centrifugo

**Test Path**: No plan status tracking - changes are immediate

**Priority**: P2 (Audit trail)

---

### Category 3: File Editor Features

#### 3.1 Accept/Reject Patch Controls (Partial)
**Main Path**: `CodeEditor.tsx:171-563`
- Diff header with file navigation (prev/next)
- Accept This File / Accept All Files dropdown
- Reject This File / Reject All Files dropdown
- Only shown when plan status is "applied"

**Test Path**: Has Commit/Discard for all pending changes but no per-file accept/reject

**Priority**: P2 (Granular change control)

---

#### 3.2 Command Menu / Command Palette (Missing)
**Main Path**:
- `CommandMenu.tsx` - Global command palette
- `CommandMenuWrapper` in `WorkspaceContent.tsx:130`
- Cmd/Ctrl+K shortcut
- Theme switching, debug panel toggle

**Test Path**: No command menu

**Priority**: P3 (Power user feature)

---

#### 3.3 Debug Panel (Missing)
**Main Path**: `DebugPanel.tsx`
- Shows selected file metadata (ID, Path)
- Shows pending content
- Toggleable via command menu
- Fixed position 25% width on right

**Test Path**: No debug panel

**Priority**: P4 (Developer tool)

---

#### 3.4 File Delete Functionality (Incomplete)
**Main Path**: `FileTree.tsx:439-452, 480-491`
- Trash icon on file hover
- Delete confirmation modal
- Protection for required files (Chart.yaml, values.yaml)

**Test Path**: Delete UI may be missing or non-functional

**Priority**: P3 (File management)

---

### Category 4: Render & Terminal Features

#### 4.1 Terminal Component for Render Output (Missing)
**Main Path**: `Terminal.tsx:17-122`
- Mac-style terminal window
- Shows `helm dependency update` command and output
- Shows `helm template` command and stderr
- Collapsible with expand/collapse buttons
- Rendered file list with checkmarks

**Test Path**: No terminal component for render output

**Priority**: P2 (Visibility into helm operations)

---

#### 4.2 Rendered Files List Display (Missing)
**Main Path**: `Terminal.tsx:103-115`
- Lists each rendered file with green checkmark
- Shows file paths for generated manifests

**Test Path**: Rendered files not visually listed

**Priority**: P3 (User feedback)

---

### Category 5: Conversion Features

#### 5.1 Conversion Progress Display (Missing)
**Main Path**: `ConversionProgress.tsx`
- Multi-step progress display
- Steps: Analyzing → Sorting → Creating → Normalizing → Simplifying → Assembly
- Status indicators (spinner/checkmark/pending)
- Real-time status updates

**Test Path**: No conversion UI or functionality

**Priority**: P3 (K8s-to-Helm conversion feature)

---

#### 5.2 Conversion File Updates (Missing)
**Main Path**: `useCentrifugo.ts:449-457`
- `handleConversionFileUpdatedMessage`
- `handleConversationUpdatedMessage`
- Updates conversion state in real-time

**Test Path**: No conversion event handling

**Priority**: P3 (Conversion feature)

---

### Category 6: Layout & Navigation Features

#### 6.1 Adaptive Layout Based on Revision (Missing)
**Main Path**: `WorkspaceContent.tsx:100, 108-127`
- New chart mode: Full-width centered chat (max-w-3xl)
- Editor mode: Split view - chat (480px) + editor (flex-1)
- Smooth transition when revision created

**Test Path**: Fixed 3-panel layout always (chat + files + editor)

**Priority**: P2 (Onboarding UX for new charts)

---

#### 6.2 New Chart Content Component (Missing)
**Main Path**: `NewChartContent.tsx`
- Special layout for revision 0 workspaces
- Centered title
- Different message rendering (`NewChartChatMessage`)
- "Create Chart" button

**Test Path**: Uses standard chat for new charts

**Priority**: P3 (Onboarding UX)

---

#### 6.3 Side Navigation (Present but may differ)
**Main Path**: `SideNav.tsx`
- Icon-based navigation (Home, Code)
- UserMenu at bottom
- Tooltips on hover

**Test Path**: Uses `EditorLayout` but side nav usage unclear

**Priority**: P4 (Navigation)

---

### Category 7: Data Loading & State

#### 7.1 Comprehensive Initial Data Load (Missing)
**Main Path**: `/workspace/[id]/page.tsx:35-40`
```typescript
const [workspace, messages, plans, renders, conversions] = await Promise.all([...])
```
Loads: workspace, messages, plans, renders, conversions

**Test Path**: `/test-ai-chat/[workspaceId]/page.tsx:32-38`
Only loads: workspace, messages

**Priority**: P2 (Data completeness)

---

#### 7.2 Plans Atom Hydration (Missing)
**Main Path**: `WorkspaceContent.tsx:63-65`
- Hydrates `plansAtom` with initial plans
- Updates via Centrifugo `plan-updated` events

**Test Path**: No plansAtom hydration

**Priority**: P2 (Plan feature dependency)

---

#### 7.3 Conversions Atom Hydration (Missing)
**Main Path**: `WorkspaceContent.tsx:66-68`
- Hydrates `conversionsAtom` with initial conversions
- Updates via Centrifugo conversion events

**Test Path**: No conversionsAtom hydration

**Priority**: P3 (Conversion feature dependency)

---

### Category 8: Feedback & Communication

#### 8.1 Feedback Modal (Missing)
**Main Path**: `FeedbackModal.tsx`
- Star rating (1-5)
- Shows user prompt, files sent, response
- Free-form feedback textarea
- Two-column layout
- Submits feedback for AI improvement

**Test Path**: No feedback modal

**Priority**: P3 (User feedback collection)

---

#### 8.2 Toast Notification System (May be missing)
**Main Path**: `use-toast.tsx`
- Global toast notifications
- Auto-dismiss
- Action buttons
- State management with reducer

**Test Path**: Usage unclear

**Priority**: P3 (User feedback)

---

## Architecture Comparison

| Aspect | Main Path (`/workspace/[id]`) | Test Path (`/test-ai-chat`) |
|--------|-------------------------------|------------------------------|
| Chat Backend | Go worker via PostgreSQL queue | Vercel AI SDK direct streaming |
| Message Action | `createChatMessageAction` → enqueueWork | `createAISDKChatMessageAction` (bypasses queue) |
| Response Handling | Go worker writes to DB, Centrifugo pushes | AI SDK streams, then persists |
| Tool Execution | Go backend handles tools directly | Next.js API calls Go HTTP endpoints |
| Plan Workflow | Full plan creation → review → apply | Direct file changes via tools |
| Initial Data | 5 parallel fetches | 2 fetches (workspace + messages) |
| Layout | Adaptive (new chart vs editor mode) | Fixed 3-panel |
| State Management | Full atom hydration | Partial atom hydration |

---

## Priority Summary

### P1 - Critical for Parity (3 items)
1. Full Plan Display UI
2. Plan Proceed/Ignore Actions
3. Role/Persona Selector

### P2 - High Value (9 items)
1. Rollback Functionality
2. Nested Chat Input for Plan Questions
3. Plan Status Tracking
4. Accept/Reject Patch Controls (per-file)
5. Terminal Component for Render Output
6. Adaptive Layout Based on Revision
7. Comprehensive Initial Data Load
8. Plans Atom Hydration
9. Conversions Atom Hydration (if conversions supported)

### P3 - Medium Value (8 items)
1. Message Cancel Button polish
2. Followup Action Buttons
3. File Delete Functionality
4. Rendered Files List Display
5. Conversion Progress Display
6. Conversion File Updates
7. New Chart Content Component
8. Feedback Modal

### P4 - Nice to Have (2 items)
1. Debug Panel
2. Side Navigation parity

---

## Recommended PR Structure

Given the scope, consider this expanded roadmap:

```
PR1.8: Page Reload Guard + Diff Visualization (as planned)
       └── ~2-3 hours, already in roadmap

PR1.9: Plan UI Foundation
       └── Plan display, proceed/ignore, status tracking
       └── ~6-8 hours, P1 priority

PR1.10: Render Tab Integration (as planned)
        └── ~4-6 hours, already in roadmap

PR1.11: Role Selector + Adaptive Layout
        └── Persona selection, new chart mode
        └── ~3-4 hours, P1/P2 priority

PR1.12: Editor Enhancement
        └── Per-file accept/reject, command menu
        └── ~4-5 hours, P2 priority

PR1.13: Plan Persistence + Terminal Output
        └── ~4-6 hours, already partially in roadmap

PR1.14: Rollback + Feedback
        └── Rollback modal, feedback modal
        └── ~3-4 hours, P2/P3 priority

PR1.15: Data Loading Parity
        └── Full initial data load, atom hydration
        └── ~2-3 hours, P2 priority
```

---

## Files to Study

| Feature Category | Key Files |
|------------------|-----------|
| Plan UI | `components/PlanChatMessage.tsx`, `atoms/workspace.ts` (planByIdAtom) |
| Role Selector | `components/ChatContainer.tsx:140-201` |
| Rollback | `components/RollbackModal.tsx`, `lib/workspace/actions/rollback.ts` |
| Terminal | `components/Terminal.tsx` |
| Command Menu | `components/CommandMenu.tsx`, `components/CommandMenuWrapper.tsx` |
| Adaptive Layout | `components/WorkspaceContent.tsx:100-127` |
| New Chart | `components/NewChartContent.tsx` |
| Feedback | `components/FeedbackModal.tsx` |
| Conversion | `components/ConversionProgress.tsx` |

---

## Success Criteria

After all features implemented:
- [ ] Plan proposals display with full UI (description, files, status)
- [ ] Users can proceed or ignore plans
- [ ] Role selector available (auto/developer/operator)
- [ ] Layout adapts for new charts vs existing workspaces
- [ ] Per-file accept/reject for pending changes
- [ ] Terminal shows helm command output
- [ ] Rollback available for reverting revisions
- [ ] Feedback modal for AI quality feedback
- [ ] Command palette with Cmd/Ctrl+K
- [ ] Full initial data loading (plans, renders, conversions)
- [ ] Test-ai-chat and workspace paths are functionally identical

---

## Open Questions

1. **Conversion Feature Scope**: Should test-ai-chat support K8s-to-Helm conversion, or is this out of scope?

2. **Plan Workflow vs Direct Changes**: The test path currently applies changes directly. Should it adopt the full plan workflow, or offer both modes?

3. **Go HTTP Endpoints**: The test path calls Go HTTP endpoints for tools. Should these endpoints be enhanced for better parity with the Go worker's capabilities?

4. **Persona in AI SDK**: How should persona/role selection be passed to the AI SDK? Currently it's embedded in the Go worker's prompt construction.

---

*This research identifies 22 additional feature gaps beyond the 5 in PR1.8, providing a comprehensive roadmap for achieving true feature parity between the two paths.*
