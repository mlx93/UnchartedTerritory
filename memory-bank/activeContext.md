# Chartsmith Active Context

**Purpose**: Track current work focus, recent changes, next steps, and active decisions.

---

## Current Phase

**Status**: PR1.7 Implementation ✅ COMPLETE

PR1.7 has been fully implemented. The AI SDK path now has:
- Real-time file updates via Centrifugo integration
- Pending changes UI indicators (yellow dots in file tree)
- Commit/Discard functionality for revision management
- Files properly display in Explorer for revision 0

---

## PR Structure & Timeline

| PR | Name | Timeline | Status |
|----|------|----------|--------|
| PR1 | AI SDK Foundation | Days 1-3 | ✅ **Complete** |
| PR1.5 | Tool Integration & Go HTTP | Days 3-4 | ✅ **Complete** |
| PR1.6 | Feature Parity | Day 5 | ✅ **Complete** |
| PR1.61 | Body Parameter Hotfix | Day 5 | ✅ **Complete** |
| PR1.65 | UI Feature Parity | Day 5 | ✅ **Complete** |
| PR1.7 Prereq | Bug Fixes for PR1.7 | Day 5 | ✅ **Complete** |
| PR1.7 | Deeper System Integration | Day 6 | ✅ **Complete** |
| PR2 | Validation Agent | Days 7-9 | **Ready for Implementation** |

---

## Completed Work Summary

### PR1.7: Deeper System Integration ✅

#### Feature 1: Centrifugo Integration (Real-time File Updates)

| Component | Description |
|-----------|-------------|
| `pkg/workspace/chart.go` | Modified `AddFileToChartPending()` to return `(string, error)` exposing the generated fileID |
| `pkg/workspace/file.go` | Added `GetFileIDByPath()` helper for str_replace operations |
| `pkg/api/handlers/editor.go` | Added `publishArtifactUpdate()` helper and Centrifugo calls after create/str_replace |
| `pkg/realtime/centrifugo.go` | Added debug logging for message publishing |
| `pkg/realtime/types/artifact-updated.go` | Event type for artifact updates (already existed) |
| `chartsmith-app/hooks/useCentrifugo.ts` | Fixed race condition in dependency array, added `handleArtifactUpdated` handler |

**Key Fixes:**
- Fixed Centrifugo connection race condition (added `publicEnv.NEXT_PUBLIC_CENTRIFUGO_ADDRESS` to useEffect deps)
- Fixed JSON field name mismatch (Go struct tags changed to camelCase: `chartId`, `revisionNumber`, `contentPending`)

#### Feature 2: Pending UI Indicators

| Component | Description |
|-----------|-------------|
| `chartsmith-app/components/FileTree.tsx` | Added yellow dot indicator for files with `contentPending` |
| `chartsmith-app/atoms/workspace.ts` | `allFilesWithContentPendingAtom` derives files with pending changes |

**Visual Indicators:**
- Yellow dot (●) next to files with pending changes
- Diff stats showing additions/deletions (e.g., `+10 / -10`)

#### Feature 3: Commit/Discard Server Actions

| Component | Description |
|-----------|-------------|
| `chartsmith-app/lib/workspace/actions/commit-pending-changes.ts` | Creates new revision promoting `content_pending` → `content` |
| `chartsmith-app/lib/workspace/actions/discard-pending-changes.ts` | Clears pending changes without creating revision |
| `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx` | Commit/Discard buttons in tab bar (right of Source/Rendered) |

**UI Location:** Buttons appear in editor tab bar when pending changes exist

#### Critical Bug Fixes

| Fix | Description | Files Changed |
|-----|-------------|---------------|
| Workspace Revision 0 Loading | Removed `> 0` check that prevented loading charts/files for revision 0 | `lib/workspace/workspace.ts` |
| Source View Content Display | Editor now shows `contentPending \|\| content` | `client.tsx` |
| RawFile Interface | Fixed snake_case to camelCase property names | `components/types.ts` |

### Previous PRs (Summary)

#### PR1.6: Feature Parity ✅
- Simplified system prompt
- Landing page with workspace creation
- FileBrowser integration
- Chat persistence actions

#### PR1.61: Body Parameter Hotfix ✅
- Fixed stale body parameter issue in AI SDK v5
- `getChatBody()` helper for fresh params at request time

#### PR1.65: UI Feature Parity ✅
- Three-panel layout (Chat → Explorer → Code)
- Monaco syntax highlighting
- Source/Rendered tabs
- ArtifactHubSearchModal integration

#### PR1.7 Prereq Fixes ✅
- Double processing bug (NON_PLAN intent)
- Content column bug (AddFileToChartPending)
- Revision number bug (0 vs 1)

---

## Key Technical Decisions (Resolved)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Real-time updates | Centrifugo WebSocket | Existing infrastructure, user-scoped channels |
| Pending changes storage | `content_pending` column | Allows diff display without creating revisions |
| Commit/Discard UI | Tab bar (right side) | Non-intrusive, contextual to editor |
| Revision creation | Only on Commit action | Batches all pending changes into single revision |

---

## Environment Setup

### Current Environment Variables

```env
# Direct API Keys (preferred)
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-xxx

# Fallback
OPENROUTER_API_KEY=sk-or-v1-xxxxx

# Configuration (defaults)
DEFAULT_AI_PROVIDER=anthropic
DEFAULT_AI_MODEL=anthropic/claude-sonnet-4-20250514

# Go backend URL
GO_BACKEND_URL=http://localhost:8080

# Centrifugo (in chartsmith-app/.env.local)
NEXT_PUBLIC_CENTRIFUGO_ADDRESS=ws://localhost:8000/connection/websocket
CENTRIFUGO_TOKEN_HMAC_SECRET=change.me
```

---

## Blocking Dependencies

### PR2 Dependencies
- [x] PR1 complete
- [x] PR1.5 complete
- [x] PR1.6 complete
- [x] PR1.7 complete
- [ ] helm CLI (v3.x) installed
- [ ] kube-score CLI (v1.16+) installed

---

## Next Actions

### NEXT: PR2 Implementation (Validation Agent)
1. Read `PRDs/PR2_Product_PRD.md` and `PRDs/PR2_Tech_PRD.md`
2. Verify helm and kube-score CLI tools are installed
3. Create `pkg/validation/` package structure
4. Create validation endpoint in Go
5. Create validateChart tool in TypeScript

### Optional Cleanup:
1. Remove debug console.log statements from `useCentrifugo.ts`
2. Remove debug logging from `centrifugo.go` and `editor.go`

---

## Sub-Agent Prompts

| Prompt | Location |
|--------|----------|
| PR1.6 (includes 1.61/1.65) | `agent-prompts/PR1.6_SUB_AGENT.md` |
| PR2 | `agent-prompts/PR2_SUB_AGENT.md` |

---

## Research Documents

| Document | Purpose |
|----------|---------|
| `docs/issues/PR1.6_POST_IMPLEMENTATION_ISSUES.md` | Issue tracking (all resolved) |
| `docs/research/2025-12-04-PR1.6-AI-SDK-STREAMING-RESEARCH.md` | Root cause analysis |
| `docs/PR1.6_COMPLETION_REPORT.md` | What was built in PR1.6 |
| `PRDs/PR1.7_IMPLEMENTATION_PLAN.md` | PR1.7 implementation plan |

---

*This document is the starting point for each work session. Last updated: Dec 5, 2025 (PR1.7 complete)*
