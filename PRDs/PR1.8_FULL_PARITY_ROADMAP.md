# PR1.8+ Full Feature Parity Roadmap

**Status**: Planning (Post-PR1.7)
**Purpose**: High-level guide for achieving 100% feature parity between `/test-ai-chat` and `/workspace/[id]`

---

## Overview

After PR1.7, the test-ai-chat path will have core functionality parity. The remaining items are polish, edge cases, and advanced features that would make the paths completely interchangeable.

---

## Feature 1: Diff Visualization

### What It Does
Side-by-side diff view showing `content` vs `content_pending` for files with pending changes.

### Main Path Reference
- `components/FileDiff.tsx` - Diff rendering component
- Uses Monaco Editor's diff view or similar

### Implementation Approach
1. Add "Show Diff" button/icon next to files with `contentPending`
2. Create `DiffViewer` component using Monaco's `DiffEditor`
3. Wire up to `selectedFileAtom` to show diff when file selected
4. Toggle between normal view and diff view

### Complexity: Medium
### Dependencies: Monaco Editor (already installed)

---

## Feature 2: Plan Persistence to Database

### What It Does
Saves AI SDK plan proposals to `workspace_plan` table so they survive page refresh and can be audited.

### Main Path Reference
- `workspace_plan` table schema
- `lib/workspace/plan.ts` - Plan CRUD operations
- Go worker creates plans via `pkg/listener/plan.go`

### Implementation Approach
1. Create `createPlanAction(workspaceId, description, changes)` server action
2. Insert into `workspace_plan` with `status: 'pending_review'`
3. Update `planProposal` tool to call this action
4. On Apply: Update status to `applied`
5. On Discard: Update status to `ignored`
6. Load persisted plans on page load (check for pending plans)

### Complexity: Medium
### Dependencies: PR1.7 Feature 4 (Plan Workflow)

---

## Feature 3: Render Tab Updates

### What It Does
Shows Helm template render output after AI makes changes. The "Rendered" tab in the code editor updates to reflect modifications.

### Main Path Reference
- `pkg/listener/render-workspace.go` - Go render job
- `workspace_rendered` and `workspace_rendered_chart` tables
- Centrifugo `render-stream` events

### Implementation Approach
1. **Option A (Simple)**: Trigger render via API after commit
   - Call existing render endpoint after `commitPendingChangesAction`
   - Poll or subscribe for render completion
   
2. **Option B (Full Integration)**: Subscribe to render events
   - Add `render-stream` handler to test-ai-chat's Centrifugo subscription
   - Update rendered content atom when events arrive
   - Show "Rendering..." indicator during render

### Complexity: Medium-High
### Dependencies: PR1.7 Feature 1 (Centrifugo), Go render infrastructure

---

## Feature 4: Conflict Resolution

### What It Does
Handles the case where multiple users/tabs edit the same file, or AI makes changes while user is editing.

### Main Path Reference
- `content_pending` conflict detection
- Warning dialogs before overwriting
- Merge strategies (not currently implemented in main path either)

### Implementation Approach
1. **Detection**: Before AI writes, check if `content_pending` already exists and differs from what AI expects
2. **Warning**: Show modal: "This file has been modified. Overwrite / Merge / Cancel"
3. **Simple Resolution**: Last-write-wins with warning (matches current main path)
4. **Future**: Three-way merge UI (significant effort)

### Complexity: Low (warnings) to Very High (merge UI)
### Dependencies: None for basic warnings

---

## Feature 5: Page Reload Guard

### What It Does
Prevents accidental loss of work when user reloads page during AI response or with uncommitted changes.

### Main Path Reference
- Browser `beforeunload` event
- Warning dialog: "You have unsaved changes"

### Implementation Approach
1. Add `useEffect` with `beforeunload` listener
2. Trigger when `hasPendingChanges === true` OR `status === 'streaming'`
3. Return standard browser warning message
4. **Bonus**: Persist `hasAutoSent` to `sessionStorage` to prevent double-send on reload

### Complexity: Low
### Dependencies: PR1.7 Feature 2 (Pending UI provides `hasPendingChanges`)

---

## Priority Recommendation

| Priority | Feature | Rationale |
|----------|---------|-----------|
| P1 | Page Reload Guard | Low effort, prevents data loss |
| P2 | Diff Visualization | High user value, medium effort |
| P3 | Render Tab Updates | Expected feature, medium effort |
| P4 | Plan Persistence | Nice-to-have, audit trail |
| P5 | Conflict Resolution | Edge case, complex to do well |

---

## Suggested PR Structure

```
PR1.8: Page Reload Guard + Diff Visualization
       └── ~2-3 hours, high impact

PR1.9: Render Tab Integration
       └── ~4-6 hours, requires Go coordination

PR1.10: Plan Persistence + Conflict Warnings
        └── ~4-6 hours, polish features
```

---

## Files to Study

| Feature | Key Files |
|---------|-----------|
| Diff | `components/CodeEditor.tsx` (Monaco diff), `atoms/workspace.ts` |
| Plan Persistence | `lib/workspace/plan.ts`, `workspace_plan` table schema |
| Render | `pkg/listener/render-workspace.go`, `hooks/useCentrifugo.ts` (render-stream handler) |
| Conflict | `pkg/workspace/file.go` (SetFileContentPending), modal components |
| Reload Guard | Browser `beforeunload` API, `sessionStorage` |

---

## Success Criteria

After all features implemented:
- [ ] Files with pending changes show diff on click
- [ ] Plan proposals persist across page refresh
- [ ] Rendered tab updates after AI changes
- [ ] Warning shown before losing unsaved work
- [ ] Concurrent edit warning (basic)
- [ ] Test-ai-chat and workspace paths are functionally identical

---

*This roadmap provides direction for detailed implementation plans. Each feature should get its own implementation plan document before coding begins.*

