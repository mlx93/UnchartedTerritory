# PR1.6: Test Path Feature Parity - Product Requirements Document

**Version**: 1.0
**PR**: PR1.6 (Feature Parity)
**Prerequisite**: PR1.5 merged
**Status**: Ready for Implementation
**Last Updated**: December 4, 2025

---

## Executive Summary

### Purpose

PR1.6 achieves feature parity between the `/test-ai-chat` path and the main `/workspace/[id]` path. After PR1.6, the AI SDK test path will be a fully functional alternative to the main path for users to interact with Chartsmith.

### Business Value

- **User Experience**: Test path becomes a complete, usable product
- **Migration Path**: Users can transition to AI SDK-powered chat
- **Validation**: Proves AI SDK approach works for Chartsmith use cases
- **Foundation**: Sets stage for PR1.7 deeper system integration

### Scope Summary

PR1.6 delivers:
1. **Working tool execution** - AI uses tools instead of outputting text
2. **Workspace creation flow** - Users can start fresh from test path
3. **File explorer integration** - Visual file tree alongside chat
4. **Chat persistence** - Messages survive page refresh
5. **CSS fixes** - Clean, readable chat UI

---

## Problem Statement

### Current State (After PR1.5)

The `/test-ai-chat` path has tools registered but several critical gaps:

| Feature | Main Path | Test Path | Gap |
|---------|-----------|-----------|-----|
| Tool execution | ✅ Works | ❌ Outputs XML text | Broken |
| Workspace creation | ✅ From homepage | ❌ Manual URL only | Missing |
| File explorer | ✅ Full tree | ❌ Not shown | Missing |
| Chat persistence | ✅ Database | ❌ In-memory only | Missing |
| CSS styling | ✅ Clean | ❌ White highlights | Broken |

### Pain Points

1. **Broken tools**: Users see `<latestSubchartVersion>...` in chat instead of getting tool results
2. **No entry point**: Cannot create new workspaces from test path
3. **Blind editing**: Cannot see files being modified
4. **Lost work**: Refreshing page loses entire conversation
5. **Unreadable text**: White highlighting obscures chat messages

### Desired State (After PR1.6)

A fully functional AI SDK chat experience:
- Tools execute and return results properly
- Users can create workspaces and start building charts
- File explorer shows real-time changes
- Chat history persists across sessions
- Clean, readable UI in both themes

---

## User Stories

### US-1: Tool Execution Works

**As a** Chartsmith user
**I want** tools to execute when the AI uses them
**So that** I get real results instead of seeing tool invocation text

**Acceptance Criteria**:
- When I ask "What files are in my chart?", AI calls getChartContext and displays file list
- When I ask to view a file, AI shows the file contents
- When I ask for subchart versions, AI returns actual version numbers
- No XML-formatted tool text appears in chat

**Priority**: P0 (Critical)

### US-2: Create Workspace from Test Path

**As a** new user
**I want** to start building a chart from the test path
**So that** I don't need to use the main path first

**Acceptance Criteria**:
- Landing page at `/test-ai-chat` shows prompt input
- Entering description and clicking submit creates workspace
- Automatically redirected to chat with new workspace
- Initial chat message reflects my description

**Priority**: P1 (High)

### US-3: See File Explorer

**As a** user editing my chart
**I want** to see the file tree alongside the chat
**So that** I can understand and navigate my chart structure

**Acceptance Criteria**:
- File explorer displays on left side of chat
- Shows chart folder structure with files
- Files update when AI creates or modifies them (via workspace refetch after tool completion)
- Can click files to see selection highlight

**Priority**: P1 (High)

### US-4: Persistent Chat History

**As a** user with an ongoing project
**I want** my chat history to persist
**So that** I can continue conversations after refreshing

**Acceptance Criteria**:
- User messages persist immediately when sent
- AI responses save when complete
- Refreshing page shows previous conversation in a "History" section above current session
- Messages display in correct order with proper formatting

**Priority**: P2 (Medium)

### US-5: Readable Chat UI

**As a** user in dark mode
**I want** clean, readable chat messages
**So that** I can easily read AI responses

**Acceptance Criteria**:
- No white highlighting over text
- Code blocks have appropriate backgrounds
- Text is readable in both light and dark modes
- Proper contrast for all elements

**Priority**: P3 (Low)

---

## Functional Requirements

### FR-1: Tool Execution

#### FR-1.1: System Prompt Simplification
- Remove verbose tool documentation from system prompt
- Keep behavioral guidelines only
- Let AI SDK provide tool schemas automatically

#### FR-1.2: Tool Response Display
- Tool invocations show with icon and tool name
- Tool results display formatted JSON
- Proper loading state during tool execution

### FR-2: Workspace Creation

#### FR-2.1: Landing Page
- Textarea for chart description
- Submit button with loading state
- Error handling for failed creation

#### FR-2.2: Creation Flow
- Call existing `createWorkspaceFromPromptAction`
- Redirect to `/test-ai-chat/{workspaceId}`
- Initialize with user's description as first message

### FR-3: File Explorer

#### FR-3.1: Layout
- Left panel (256px width) for file tree
- Right panel for chat interface
- Responsive split view

#### FR-3.2: Data Integration
- Hydrate Jotai atoms with workspace data
- Use existing `FileBrowser` component
- Real-time updates when files change

### FR-4: Chat Persistence

#### FR-4.1: User Messages
- Save to `workspace_chat` table on send
- Track message ID for response linking
- Create new `createAISDKChatMessageAction` to skip Go backend intent processing

#### FR-4.2: AI Responses
- Save response when AI completes
- New `updateChatMessageResponseAction` server action
- Extract text from AI message parts

#### FR-4.3: Loading History
- Fetch messages on page load
- Display in chat UI before new messages
- Maintain conversation context

### FR-5: CSS Fixes

#### FR-5.1: Remove Prose Conflicts
- Remove `prose` classes from message rendering
- Add explicit styling for code blocks
- Ensure dark mode compatibility

---

## Non-Functional Requirements

### NFR-1: Performance

| Metric | Target |
|--------|--------|
| Tool execution latency | < 2s |
| File explorer render | < 100ms |
| Message persistence | < 500ms |
| Page load with history | < 1s |

### NFR-2: Usability

- Clear visual feedback for all actions
- Consistent styling with main path
- Intuitive file explorer navigation
- Readable text in all themes

### NFR-3: Reliability

- Messages never lost during normal operation
- Graceful degradation if Go backend unavailable
- Clear error messages for failures
- No data corruption on concurrent edits

### NFR-4: Compatibility

- Works with all supported AI providers
- Compatible with existing workspace structure
- Uses same database schema as main path
- Works with current authentication system

---

## Testing Requirements

### Manual Testing Scenarios

| Scenario | Steps | Expected Result |
|----------|-------|-----------------|
| Tool execution | Ask "What's the latest redis version?" | Returns actual version (e.g., "18.x.x") |
| Workspace creation | Enter prompt, click submit | Redirects to new workspace |
| File explorer | Navigate to workspace | File tree displays |
| Real-time updates | Ask AI to create file | File appears in explorer |
| Persistence | Send message, refresh | Message still visible |
| Dark mode | Toggle theme | No highlighting issues |

### Automated Testing

- Unit tests for new server action
- Integration test for workspace creation flow
- Component tests for file explorer integration

---

## Success Criteria

### Must Pass (P0-P1)

- [ ] Tools execute properly (no XML text output)
- [ ] Workspace creation works from landing page
- [ ] File explorer displays and updates

### Should Pass (P2)

- [ ] Chat messages persist across refresh
- [ ] History loads on page mount

### Nice to Have (P3)

- [ ] CSS issues fully resolved
- [ ] Perfect theme compatibility

---

## Priority Classification

### MUST HAVE (MVP)

| Priority | Feature | Rationale |
|----------|---------|-----------|
| 1 | Tool execution fix | Core functionality broken |
| 2 | Workspace creation | No entry point without it |
| 3 | File explorer | Users need to see changes |

### SHOULD HAVE

| Priority | Feature | Rationale |
|----------|---------|-----------|
| 4 | Chat persistence | Better user experience |

### NICE TO HAVE

| Priority | Feature | Rationale |
|----------|---------|-----------|
| 5 | CSS fixes | Cosmetic improvement |

---

## Dependencies

### External Dependencies
- PR1.5 merged and deployed
- Go backend running on port 8080
- PostgreSQL database accessible
- AI provider API available

### Internal Dependencies
- Existing `createWorkspaceFromPromptAction`
- Existing `FileBrowser` component
- Existing `workspace_chat` table schema
- Existing Jotai atom structure

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| System prompt change breaks something | Medium | High | Test all tool types thoroughly |
| File explorer doesn't integrate cleanly | Low | Medium | Use existing component patterns |
| Persistence conflicts with AI SDK state | Medium | Medium | Design hybrid approach carefully |
| CSS changes affect main path | Low | Low | Scope changes to test path only |

---

## Out of Scope (PR1.6)

The following are explicitly NOT included:

1. **Revision tracking** - Creating revisions from AI SDK tool calls
2. **Centrifugo integration** - Real-time updates via WebSocket
3. **Plan workflow** - Multi-file Plans from AI SDK
4. **Export functionality** - Exporting chart from test path
5. **Role selector** - Different AI personas

These are addressed in PR1.7.

---

## Appendix: Feature Comparison

### After PR1.6 Completion

| Feature | Main Path | Test Path | Status |
|---------|-----------|-----------|--------|
| Tool execution | ✅ | ✅ | Parity |
| Workspace creation | ✅ | ✅ | Parity |
| File explorer | ✅ | ✅ | Parity |
| Chat persistence | ✅ | ✅ | Parity |
| CSS styling | ✅ | ✅ | Parity |
| Revision tracking | ✅ | ❌ | PR1.7 |
| Real-time Centrifugo | ✅ | ❌ | PR1.7 |
| Plan workflow | ✅ | ❌ | PR1.7 |

---

## Related Documents

| Document | Purpose |
|----------|---------|
| `PR1.6_Tech_PRD.md` | Technical specification |
| `PR1.6_SUB_AGENT.md` | Implementation agent prompt |
| `docs/research/2025-12-04-PR1.6-FEATURE-PARITY-ANALYSIS.md` | Research findings |
| `docs/research/2025-12-04-PR1.6-PRD-GAPS-ANALYSIS.md` | Implementation gotchas and fixes |
| `PR1.7_Product_PRD.md` | Deferred features specification |

---

*Document End - PR1.6: Test Path Feature Parity Product PRD*
