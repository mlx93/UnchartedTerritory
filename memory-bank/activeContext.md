# Chartsmith Active Context

**Purpose**: Track current work focus, recent changes, next steps, and active decisions.

---

## Current Phase

**Status**: PR1.6 + PR1.61 + PR1.65 ✅ COMPLETE → Ready for PR1.7

All feature parity and UI parity work is complete. The test-ai-chat path now matches the main workspace path in functionality and appearance.

---

## PR Structure & Timeline

| PR | Name | Timeline | Status |
|----|------|----------|--------|
| PR1 | AI SDK Foundation | Days 1-3 | ✅ **Complete** |
| PR1.5 | Tool Integration & Go HTTP | Days 3-4 | ✅ **Complete** |
| PR1.6 | Feature Parity | Day 5 | ✅ **Complete** |
| PR1.61 | Body Parameter Hotfix | Day 5 | ✅ **Complete** |
| PR1.65 | UI Feature Parity | Day 5 | ✅ **Complete** |
| PR1.7 | Deeper System Integration | TBD | **Ready for Implementation** |
| PR2 | Validation Agent | Days 4-6 | Pending PR1.7 |

---

## Completed Work Summary

### PR1.6: Feature Parity ✅

| Component | Description |
|-----------|-------------|
| `lib/ai/prompts.ts` | Simplified system prompt (removes tool docs competition) |
| `app/test-ai-chat/page.tsx` | Landing page with workspace creation |
| `app/test-ai-chat/[workspaceId]/page.tsx` | Server component for data fetching |
| `app/test-ai-chat/[workspaceId]/client.tsx` | Client component with FileBrowser, persistence, CSS fixes |
| `lib/workspace/actions/create-ai-sdk-chat-message.ts` | Server action for user message persistence |
| `lib/workspace/actions/update-chat-message-response.ts` | Server action for AI response persistence |

### PR1.61: Body Parameter Hotfix ✅

**Root Cause Fixed**: AI SDK v5 `useChat` body parameter was captured at initialization and became stale.

**Solution Applied**:
- Removed `body` from `useChat` hook configuration
- Added `getChatBody()` helper with `useCallback` for fresh params
- Pass body at request-time via `sendMessage()` second argument
- Fixed auto-send from URL to use request-level body
- Fixed manual send to use request-level body

**Issues Resolved**:
- ✅ Issue #1: No streaming on first message
- ✅ Issue #2: Tool hallucination (AI uses fake XML)
- ✅ Issue #5: First message vs subsequent behavior

### PR1.65: UI Feature Parity ✅

| Feature | Status |
|---------|--------|
| Three-panel layout (Chat LEFT → Explorer MIDDLE → Code RIGHT) | ✅ |
| Code editor panel with Monaco syntax highlighting | ✅ |
| Source/Rendered tabs for file viewing | ✅ |
| Loading states and visual feedback | ✅ |
| Landing page polish with upload options | ✅ |
| ArtifactHubSearchModal integration | ✅ |
| Consistent styling matching main workspace | ✅ |

---

## Documented Deferrals (PR1.7)

1. **Revision tracking** - Creating revisions from AI SDK tool calls requires batching design
2. **Centrifugo integration** - Real-time file updates will replace refetch pattern
3. **Plan workflow** - Multi-file Plans from AI SDK require Go integration

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
| **Body parameter** | Request-level via sendMessage() | Prevents stale data issues |
| **Transport** | Default (no TextStreamChatTransport) | Required for tool support |

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
- [x] PR1.61 complete
- [x] PR1.65 complete

### PR2 Dependencies
- [x] PR1 complete
- [x] PR1.5 complete
- [x] PR1.6 complete (optional but recommended)
- [ ] helm CLI (v3.x) installed
- [ ] kube-score CLI (v1.16+) installed

---

## Next Actions

### NEXT: PR1.7 Implementation
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
| `PRDs/PR1.61_IMPLEMENTATION_PLAN.md` | Body parameter fix plan |
| `PRDs/PR1.65_UI_PARITY_PLAN.md` | UI parity implementation |

---

*This document is the starting point for each work session. Last updated: Dec 5, 2025 (PR1.6, PR1.61, PR1.65 all complete)*
