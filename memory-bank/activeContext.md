# Chartsmith Active Context

**Purpose**: Track current work focus, recent changes, next steps, and active decisions.

---

## Current Phase

**Status**: PR1.6 ISSUES IDENTIFIED → Fix Required Before PR1.7

PR1.6 (Feature Parity) infrastructure is complete, but **critical streaming issues** were discovered:
- First message from entry point doesn't stream
- Tool hallucination (AI uses fake XML instead of real tools)
- Root cause identified: **stale `body` parameter in `useChat`**

**Research complete**: See `docs/research/2025-12-04-PR1.6-AI-SDK-STREAMING-RESEARCH.md`

---

## PR Structure & Timeline

| PR | Name | Timeline | Status |
|----|------|----------|--------|
| PR1 | AI SDK Foundation | Days 1-3 | ✅ **Complete** |
| PR1.5 | Tool Integration & Go HTTP | Days 3-4 | ✅ **Complete** |
| PR1.6 | Feature Parity | Day 5 | ✅ **Complete** |
| PR1.7 | Deeper System Integration | TBD | **Ready for Implementation** |
| PR2 | Validation Agent | Days 4-6 | Pending PR1.7 |

---

## PR1.6 Completion Summary

### What Was Built

| Component | Description |
|-----------|-------------|
| `lib/ai/prompts.ts` | Simplified system prompt (removes tool docs competition) |
| `app/test-ai-chat/page.tsx` | Landing page with workspace creation |
| `app/test-ai-chat/[workspaceId]/page.tsx` | Server component for data fetching |
| `app/test-ai-chat/[workspaceId]/client.tsx` | Client component with FileBrowser, persistence, CSS fixes |
| `lib/workspace/actions/create-ai-sdk-chat-message.ts` | Server action for user message persistence |
| `lib/workspace/actions/update-chat-message-response.ts` | Server action for AI response persistence |

### Key Implementation Details

- **System Prompt Fix**: Removed verbose tool documentation that competed with AI SDK's automatic tool schema passing
- **Server/Client Split**: Test page now follows server/client component pattern like main workspace
- **Jotai Hydration**: Only `workspaceAtom` is set (derived atoms update automatically)
- **File Explorer Updates**: Workspace refetch after tool completion (Centrifugo deferred to PR1.7)
- **Chat Persistence**: New AI SDK-specific actions that bypass Go intent processing
- **CSS Fix**: Removed prose classes, added explicit theme-aware code styling

### Documented Deferrals (PR1.7)

1. **Revision tracking** - Creating revisions from AI SDK tool calls requires batching design
2. **Centrifugo integration** - Real-time file updates will replace refetch pattern
3. **Plan workflow** - Multi-file Plans from AI SDK require Go integration

### PR1.6 Completion Report

- `docs/PR1.6_COMPLETION_REPORT.md` - Full implementation details

---

## Current Work Focus: PR1.6 Hotfix

### BLOCKER: AI SDK Body Parameter Issue

**Root Cause**: The `body` parameter in `useChat` is captured at initialization and becomes stale. When first message fires, `workspaceId` may be undefined, causing tools not to be created on the server.

### Required Fix

**File**: `chartsmith-app/app/test-ai-chat/[workspaceId]/client.tsx`

1. Remove `body` from `useChat` config
2. Pass body in each `sendMessage()` call as second argument:
```typescript
sendMessage(
  { text: prompt },
  { body: { workspaceId: workspace.id, revisionNumber, provider, model } }
);
```

### Issues Addressed by Fix
- Issue #1: No streaming on first message
- Issue #2: Tool hallucination (wrong answers)
- Issue #5: First message vs subsequent behavior

### Remaining Issues (After Fix)
- Issue #4: Conversation order reversal - needs separate investigation

### Sub-Agent Prompts

- PR1.7 ready at: `agent-prompts/PR1.6_SUB_AGENT.md` (contains PR1.7 readiness notes)
- PR2 ready at: `agent-prompts/PR2_SUB_AGENT.md`

---

## Key Technical Decisions (Resolved)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Migration approach | Hybrid: New system parallel | Meets "replace" requirement with safety net |
| Go LLM responsibility | None after PR1.5 | Clean separation: Node thinks, Go executes |
| Tool count | **4 tools** (not 6) | Workspace creation handled by homepage flow |
| Go communication | HTTP endpoints (port 8080) | Synchronous tool execution |
| Tool → Go pattern | All 4 tools call Go | Consistent architecture, avoids bundling issues |
| Default model | Claude Sonnet 4 | Anthropic's recommended model |
| System prompt style | Minimal, behavior-focused | Let AI SDK provide tool schemas |
| Chat persistence | AI SDK-specific actions | Bypass Go intent processing |

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
```

---

## Blocking Dependencies

### PR1.7 Dependencies
- [x] PR1 complete
- [x] PR1.5 complete
- [x] PR1.6 complete

### PR2 Dependencies
- [x] PR1 complete
- [x] PR1.5 complete
- [x] PR1.6 complete (optional but recommended)
- [ ] helm CLI (v3.x) installed
- [ ] kube-score CLI (v1.16+) installed

---

## Next Actions

### IMMEDIATE: PR1.6 Hotfix
1. Apply request-level body fix to `client.tsx`
2. Test first message flow (redirect from landing page)
3. Verify tools execute correctly (ask for subchart version)
4. Investigate Issue #4 (conversation order) separately

### After Hotfix: PR1.7
1. Read `PRDs/PR1.7_Product_PRD.md` and `PRDs/PR1.7_Tech_PRD.md`
2. Design revision batching strategy for AI SDK tool calls
3. Integrate Centrifugo for real-time file updates
4. Implement Plan workflow support

### For PR2:
1. Read `PRDs/PR2_Product_PRD.md` and `PRDs/PR2_Tech_PRD.md`
2. Verify helm and kube-score CLI tools are installed
3. Create `pkg/validation/` package structure
4. Create validation endpoint in Go
5. Create validateChart tool in TypeScript

---

## Research Documents

| Document | Purpose |
|----------|---------|
| `docs/issues/PR1.6_POST_IMPLEMENTATION_ISSUES.md` | Issue tracking with root cause |
| `docs/research/2025-12-04-PR1.6-AI-SDK-STREAMING-RESEARCH.md` | Full research findings |
| `docs/PR1.6_COMPLETION_REPORT.md` | What was built in PR1.6 |

---

*This document is the starting point for each work session. Last updated: Dec 4, 2025 (PR1.6 issues identified, fix documented)*
