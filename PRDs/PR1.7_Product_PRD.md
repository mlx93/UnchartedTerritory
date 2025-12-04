# PR1.7: Deep System Integration - Product Requirements Document

**Version**: 1.0
**PR**: PR1.7 (System Integration)
**Prerequisite**: PR1.6 merged
**Status**: Preliminary (Architectural Focus)
**Last Updated**: December 4, 2025

---

## Executive Summary

### Purpose

PR1.7 integrates the AI SDK test path deeply into Chartsmith's existing infrastructure, enabling revision tracking, real-time updates, and plan workflows. These features require significant architectural decisions that were deferred from PR1.6.

### Business Value

- **Change History**: Users can rollback AI modifications
- **Real-Time UX**: File explorer updates instantly when AI makes changes
- **Review Workflow**: Users can review multi-file changes before applying
- **Production Ready**: Test path becomes suitable for production use

### Scope Summary

PR1.7 delivers:
1. **Revision tracking** - Create revisions from AI SDK file modifications
2. **Centrifugo real-time** - Live updates when AI modifies files
3. **Plan workflow** - Multi-file change proposals with review

---

## Problem Statement

### Current State (After PR1.6)

The test path has feature parity for basic operations but lacks deeper system integration:

| Feature | Main Path | Test Path (PR1.6) | Gap |
|---------|-----------|-------------------|-----|
| Tool execution | ✅ | ✅ | Parity |
| Workspace creation | ✅ | ✅ | Parity |
| File explorer | ✅ | ✅ | Parity |
| Chat persistence | ✅ | ✅ | Parity |
| **Revision tracking** | ✅ | ❌ | **Missing** |
| **Real-time updates** | ✅ | ❌ | **Missing** |
| **Plan review** | ✅ | ❌ | **Missing** |

### Pain Points

1. **No change history**: AI modifications overwrite files without creating revisions
2. **No rollback**: Users cannot undo AI changes
3. **Stale UI**: File explorer requires manual refresh after AI changes
4. **No review**: Complex changes happen immediately without user approval

### Desired State (After PR1.7)

Full integration with Chartsmith's existing systems:
- AI changes tracked in revision history
- File explorer updates in real-time
- Complex changes proposed for review before applying
- Test path production-ready for migration

---

## User Stories

### US-1: Track AI Changes in Revisions

**As a** Chartsmith user
**I want** AI modifications tracked in revision history
**So that** I can see what changed and rollback if needed

**Acceptance Criteria**:
- When AI modifies files, changes go to "pending" state
- UI shows indicator when pending changes exist
- "Commit Changes" button creates new revision
- "Discard Changes" button reverts pending modifications
- Revision history shows AI-created revisions

**Priority**: P1 (High)

### US-2: Real-Time File Updates

**As a** user watching the AI work
**I want** the file explorer to update immediately
**So that** I can see changes as they happen

**Acceptance Criteria**:
- File appears in explorer immediately when AI creates it
- File content updates when AI modifies it
- No page refresh required
- Works across browser tabs

**Priority**: P1 (High)

### US-3: Review Multi-File Changes

**As a** user requesting complex changes
**I want** to review proposed changes before they're applied
**So that** I can approve or request modifications

**Acceptance Criteria**:
- User can request "Plan this change"
- AI proposes changes without executing
- UI shows proposed file changes with preview
- "Apply Changes" executes all proposed changes
- "Discard" cancels the proposal

**Priority**: P2 (Medium)

---

## Functional Requirements

### FR-1: Revision Tracking

#### FR-1.1: Pending Changes Model
- AI changes write to `content_pending` column (existing)
- `content` column remains unchanged until commit
- UI differentiates pending vs committed content

#### FR-1.2: Commit Flow
- "Commit Changes" button creates new revision
- All pending changes copied to new revision
- `content_pending` cleared after commit
- Workspace current revision updated

#### FR-1.3: Discard Flow
- "Discard Changes" clears all `content_pending`
- No revision created
- Files revert to committed state

#### FR-1.4: Revision Display
- Revision selector shows AI-created revisions
- Each revision labeled with creation type
- Rollback works normally

### FR-2: Centrifugo Real-Time

#### FR-2.1: Event Publishing
- Go `textEditor` endpoint publishes after file modification
- Event includes file ID, path, content, pending content
- Publishes to workspace channel

#### FR-2.2: Event Handling
- Test path subscribes to workspace channel
- `artifact-updated` events update workspace atom
- File explorer re-renders automatically

#### FR-2.3: Connection Management
- Reconnection on disconnect
- Event replay on reconnection
- Connection status indicator (optional)

### FR-3: Plan Workflow

#### FR-3.1: Plan Request Detection
- User message containing "plan" triggers plan mode
- AI uses `planProposal` tool instead of direct changes
- System prompt updated with planning instructions

#### FR-3.2: Plan Display
- Proposed changes shown in card UI
- Description rendered as Markdown
- File list with action icons (create/modify/delete)
- Preview capability for changes

#### FR-3.3: Plan Actions
- "Apply Changes" executes all proposed changes
- Changes applied via textEditor tool
- Single revision created for all changes
- "Discard" cancels proposal

---

## Non-Functional Requirements

### NFR-1: Performance

| Metric | Target |
|--------|--------|
| Centrifugo event latency | < 100ms |
| Commit revision operation | < 2s |
| Plan proposal display | < 500ms |

### NFR-2: Reliability

- No data loss during commit operation
- Atomic revision creation
- Graceful handling of Centrifugo disconnect
- Recovery of pending changes on reconnection

### NFR-3: Consistency

- Revision model matches main path
- Event format compatible with existing handlers
- Plan UI consistent with existing PlanChatMessage

---

## Feature Priority

### Priority 1: Revision Tracking

**Rationale**: Foundation for change management. Without revisions, users have no safety net for AI modifications. This should be implemented first.

**Scope**:
- Pending changes indicator
- Commit Changes button and server action
- Discard Changes button
- Revision creation with AI label

### Priority 2: Centrifugo Real-Time

**Rationale**: Significantly improves UX. Depends on revision tracking being stable.

**Scope**:
- Go endpoint Centrifugo publishing
- useCentrifugo integration in test path
- Artifact update handling

### Priority 3: Plan Workflow

**Rationale**: Advanced feature for power users. Can be implemented after core features stable.

**Scope**:
- planProposal tool
- Plan display component
- Apply/Discard actions
- Optional: Plan persistence

---

## Architectural Decisions

### Decision 1: Revision Trigger

**Question**: When should revisions be created?

**Options**:
- A) Per tool call (too granular)
- B) Per conversation (hard to detect end)
- C) Explicit user action (recommended)
- D) Hybrid with pending state (recommended)

**Decision**: Option D - Use `content_pending` for AI changes, explicit "Commit" for revision creation.

**Rationale**: Balances user control with change safety. Uses existing database pattern.

### Decision 2: Centrifugo Publishing Location

**Question**: Where should Centrifugo events be published?

**Options**:
- A) From Go endpoint (recommended)
- B) From TypeScript after tool result
- C) Polling instead of WebSocket

**Decision**: Option A - Publish from Go endpoint after file modification.

**Rationale**: Ensures all clients receive updates, matches existing pattern, single source of truth.

### Decision 3: Plan Persistence

**Question**: Should AI SDK plans persist to database?

**Options**:
- A) Persist to workspace_plan table
- B) Client-side only
- C) New ai_sdk_plan table

**Decision**: Option B for MVP, Option A as enhancement.

**Rationale**: Start simple, add persistence if needed for audit/review workflows.

---

## Success Criteria

### MVP Success (Required)

- [ ] Pending changes indicator shows when AI modifies files
- [ ] Commit Changes creates revision and clears pending
- [ ] Discard Changes clears pending without revision
- [ ] File explorer updates in real-time on AI changes

### Full Success (Stretch)

- [ ] Plan workflow enables multi-file review
- [ ] Plans can be approved or discarded
- [ ] AI auto-detects when planning is appropriate

---

## Dependencies

### External Dependencies
- PR1.6 merged and deployed
- Go Centrifugo client functional
- Database schema supports `content_pending`

### Internal Dependencies
- Existing revision creation pattern
- useCentrifugo hook
- Jotai workspace atoms

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Revision explosion from frequent commits | Medium | Medium | Clear UX guidance, batch commits |
| Centrifugo latency issues | Low | Medium | Fallback to polling if needed |
| Plan complexity confuses users | Medium | Low | Clear documentation, optional feature |
| Pending changes lost on navigation | Medium | High | Warning dialog, auto-save |

---

## Out of Scope (PR1.7)

The following are explicitly NOT included:

1. **Automatic revision creation** - Requires conversation end detection
2. **Plan sharing** - Sharing proposed plans with team members
3. **Diff visualization** - Side-by-side diff view of changes
4. **Conflict resolution** - Handling concurrent edits
5. **Plan templates** - Reusable plan patterns

---

## Related Documents

| Document | Purpose |
|----------|---------|
| `PR1.7_Tech_PRD.md` | Technical specification |
| `PR1.6_Product_PRD.md` | PR1.6 specification (prerequisite) |
| `lib/workspace/workspace.ts` | Revision patterns |
| `hooks/useCentrifugo.ts` | Real-time patterns |
| `components/PlanChatMessage.tsx` | Plan UI patterns |

---

*Document End - PR1.7: Deep System Integration Product PRD*
