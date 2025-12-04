# PR1.65: UI Feature Parity Plan

**Version**: 1.2
**Status**: BLOCKED - See Prerequisites (Root Cause Identified)
**Prerequisite**: PR1.6 Complete + Critical Issues Resolved
**Estimated Effort**: 7-10 hours

---

## ⚠️ BLOCKING ISSUES (Root Cause Identified)

Before implementing UI parity, the following critical streaming/tool issues from PR1.6 must be resolved:

**See**: [`docs/issues/PR1.6_POST_IMPLEMENTATION_ISSUES.md`](../docs/issues/PR1.6_POST_IMPLEMENTATION_ISSUES.md)

| Issue | Description | Severity | Root Cause |
|-------|-------------|----------|------------|
| No streaming on first message | Redirect from `/test-ai-chat` doesn't stream | P0 | Stale `body` in useChat |
| Tool hallucination | AI hallucinates tools instead of calling them | P0 | `workspaceId` undefined on first request |
| Wrong answers | Returns outdated version (16.2.7 vs 18.1.13) | P0 | Tools not created (no workspaceId) |
| Conversation order reversed | Messages appear in wrong order on refresh | P1 | TBD |

**Root Cause**: The `body` parameter in `useChat` is captured at initialization and becomes stale. Fix by passing body at request-time via `sendMessage()` second argument.

**Fix Applied In**: `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

---

## Objective

Achieve full visual and functional parity between `/test-ai-chat` and `/workspace/[id]` paths so users have a consistent experience regardless of which path they use.

---

## Current State (After PR1.6)

| Feature | Main Path | Test Path | Gap |
|---------|-----------|-----------|-----|
| Layout | Chat LEFT → Explorer MIDDLE → Code RIGHT | Explorer LEFT → Chat RIGHT | ❌ Different |
| Code Editor Panel | Full syntax-highlighted viewer | **Missing** | ❌ Missing |
| Source/Rendered Tabs | Toggle view modes | Missing | ❌ Missing |
| Loading Spinner | "Thinking" bubble with spinner | No visual feedback | ❌ Missing |
| Diff Indicators | "Showing X/Y files with diffs" | Missing | ❌ Missing |
| Jump to Latest | Button in chat | Missing | ❌ Missing |
| Left Sidebar | Home icon, code icon | Missing | ❌ Missing |
| Landing Page | Beautiful 3D background, gradient text | Simple dark page | ❌ Different |
| Upload Buttons | Helm chart, K8s manifests, Artifact Hub | Just "Start Building" | ❌ Missing |

---

## Implementation Phases

### Phase 1: Loading States & Visual Feedback (P0) - 30 min

**Goal**: Improve loading indicators when AI is processing.

**Files to Modify**:
- `app/test-ai-chat/[workspaceId]/client.tsx`

**Note**: Test-ai-chat already has partial loading state implementation at `client.tsx:511-529`. This phase enhances the existing patterns rather than building from scratch.

**Existing Implementation**:
- Thinking indicator exists for `status === "submitted"` (lines 511-529)
- `pendingPrompt` state shows prompt immediately (lines 374-391)
- Button loading state with `Loader2` (line 573)

**Changes**:
1. Add cancel button to thinking state (like main path `ChatMessage.tsx:238-255`)
2. Match exact styling from main path's `LoadingSpinner` component (`ChatMessage.tsx:40-48`)
3. Ensure consistent loading feedback across all states

**Code Pattern** (from `ChatMessage.tsx:40-48`):
```typescript
<div className="flex items-center gap-2">
  <div className="flex-shrink-0 animate-spin rounded-full h-3 w-3 border border-t-transparent border-primary"></div>
  <div className={`text-xs ${theme === "dark" ? "text-gray-400" : "text-gray-500"}`}>{message}</div>
</div>
```

---

### Phase 2: Three-Panel Layout (P0) - 3-4 hours

**Goal**: Match main path's layout: Chat LEFT → Explorer MIDDLE → Code Editor RIGHT

**Files to Modify**:
- `app/test-ai-chat/[workspaceId]/client.tsx` (significant restructure)

**Complexity Note**: This phase requires restructuring most of `client.tsx`. The current layout is fundamentally different - this is not a simple CSS change.

**Current Layout** (`client.tsx:207-581`):
```
┌─────────────┬────────────────────────────┐
│  Explorer   │         Chat               │
│   (left)    │        (right)             │
│  (256px)    │       (flex-1)             │
└─────────────┴────────────────────────────┘
```

**Target Layout** (from `WorkspaceContent.tsx:106-129`):
```
┌────────────────┬─────────────┬───────────────────┐
│     Chat       │  Explorer   │   Code Editor     │
│    (left)      │  (middle)   │     (right)       │
│   (480px)      │  (260px)    │    (flex-1)       │
└────────────────┴─────────────┴───────────────────┘
```

**Implementation** (reference `WorkspaceContainerClient.tsx:74-82`):
1. Restructure flex layout to 3 panels with fixed widths
2. Move chat panel to left (480px fixed)
3. Move file explorer to middle (260px fixed)
4. Add code editor panel on right (flex-1)
5. Add 1px dividers between panels with theme-aware colors

**Key Patterns to Copy**:
- Panel widths: `w-[480px]`, `w-[260px]`, `flex-1 min-w-0`
- Divider: `w-px bg-dark-border` / `bg-gray-200`
- Container height: `h-[calc(100vh-3.5rem)]`

---

### Phase 3: Code Editor Panel (P1) - 2-3 hours

**Goal**: Show file contents when clicking files in explorer.

**Files to Create/Modify**:
- `app/test-ai-chat/[workspaceId]/client.tsx`

**Implementation**:
1. Import Monaco Editor from `@monaco-editor/react`
2. Connect `selectedFileAtom` to display file on click (currently not imported in test-ai-chat)
3. Display file content with syntax highlighting
4. Add Source/Rendered tabs (pattern at `WorkspaceContainerClient.tsx:84-113`)

**Component to Reuse**:
- `components/CodeEditor.tsx` - Full Monaco editor with diff support (lines 849-860 for basic editor)
- Or use direct Monaco `Editor` import for simpler read-only view (like `WorkspaceContainerClient.tsx:156-185`)

**Key Atoms** (all exist in `atoms/workspace.ts`):
```typescript
import { selectedFileAtom, editorContentAtom, editorViewAtom } from "@/atoms/workspace";
```

**Note**: `selectedFileAtom` is currently NOT used in test-ai-chat. FileBrowser component is rendered but clicking files has no effect.

---

### Phase 4: Left Sidebar (P2) - 1 hour

**Goal**: Add home icon and code toggle icon to left side.

**Files to Modify**:
- `app/test-ai-chat/[workspaceId]/client.tsx`
- Or create `components/test-ai-chat/Sidebar.tsx`

**Implementation**:
1. Add vertical sidebar on far left
2. Add home icon (link to `/test-ai-chat`)
3. Add code/bracket icon (toggle code panel visibility)

---

### Phase 5: Landing Page Polish (P2) - 1-2 hours

**Goal**: Match main homepage styling.

**Files to Modify**:
- `app/test-ai-chat/page.tsx`

**Changes**:
1. Add 3D cube background image
2. Add gradient "Welcome to ChartSmith" or "AI SDK Test Chat" heading
3. Add upload buttons: "Upload Helm chart", "Upload K8s manifests", "Artifact Hub"
4. Match styling and animations

---

### Phase 6: Additional Polish (P3) - 1 hour

**Goal**: Small UI enhancements.

**Items**:
1. "Jump to latest" button in chat
2. Diff indicators ("Showing X/Y files with diffs")
3. User avatar in chat input area
4. Match exact colors, spacing, borders

---

## Priority Order

| Priority | Phase | Effort | Rationale |
|----------|-------|--------|-----------|
| P0 | Phase 1: Loading States | 30min | Enhancement only - patterns already exist |
| P0 | Phase 2: Three-Panel Layout | 3-4h | Major restructure of client.tsx |
| P1 | Phase 3: Code Editor | 2-3h | Core functionality gap |
| P2 | Phase 4: Left Sidebar | 1h | Navigation consistency |
| P2 | Phase 5: Landing Page | 1-2h | First impression |
| P3 | Phase 6: Polish | 1h | Nice to have |

---

## Files Reference

**Main Path Files to Study**:
- `components/WorkspaceContent.tsx` - Main layout (lines 106-129)
- `components/WorkspaceContainerClient.tsx` - Three-panel structure (lines 74-82)
- `components/ChatContainer.tsx` - Chat with loading states (lines 116-219)
- `components/CodeEditor.tsx` - Monaco editor with diff support (lines 849-860 for basic, 811-828 for diff)
- `components/ChatMessage.tsx` - Loading spinner patterns (lines 40-48, 238-255)
- `app/page.tsx` - Landing page styling

**State Management**:
- `atoms/workspace.ts` - All workspace atoms (workspaceAtom, selectedFileAtom, editorViewAtom, etc.)

---

## Success Criteria

- [ ] Loading spinner shows during AI processing
- [ ] Layout matches main path (3-panel)
- [ ] Clicking file shows content in code editor
- [ ] Source/Rendered tabs work
- [ ] Landing page looks polished
- [ ] Visual inspection shows no obvious differences

---

## Out of Scope

- Centrifugo real-time (PR1.7)
- Revision tracking (PR1.7)
- Plan workflow (PR1.7)
- Full WorkspaceContent component reuse (too complex)

---

*Ready for implementation after PR1.6 is verified working.*

