# Chartsmith Migration Progress

**Purpose**: Track what works, what's left to build, current status, and known issues.

---

## Overall Status

| Phase | Status | Notes |
|-------|--------|-------|
| Research & Analysis | ✅ Complete | Codebase analyzed, architecture understood |
| PRD Creation | ✅ Complete | All PRDs written and validated |
| Architecture Decisions | ✅ Complete | 8 key decisions documented |
| PR1 Implementation | ✅ **Complete** | AI SDK foundation working |
| PR1.5 Implementation | ✅ **Complete** | Go HTTP server + 4 tools working |
| PR1.6 Implementation | ✅ **Complete** | Test path feature parity |
| PR1.61 Implementation | ✅ **Complete** | Body parameter hotfix |
| PR1.65 Implementation | ✅ **Complete** | UI feature parity |
| PR1.7 Prereq Fixes | ✅ **Complete** | Bug fixes before PR1.7 |
| PR1.7 Implementation | ✅ **Complete** | Deeper system integration |
| PR2.0 Implementation | ✅ **Complete** | AI SDK reintegration into main path |
| PR2 Implementation | 🔲 Ready to Start | Validation agent |
| PR3.0 Parity Plan | ✅ Ready | Plan workflow parity (Option A) |

---

## What Already Works

### PR2.0 Deliverables (All Working ✅)

- [x] `lib/chat/messageMapper.ts` - UIMessage ↔ Message format conversion
- [x] `lib/chat/__tests__/messageMapper.test.ts` - 33 unit tests (all passing)
- [x] `hooks/useAISDKChatAdapter.ts` - Adapter bridging AI SDK to existing patterns
- [x] `hooks/useLegacyChat.ts` - Legacy Go worker path wrapper
- [x] Feature flag `NEXT_PUBLIC_USE_AI_SDK_CHAT` for rollout/rollback
- [x] Persona propagation (auto/developer/operator) to `/api/chat`
- [x] Status flag mapping (submitted/streaming/ready)
- [x] Cancel support with partial response persistence
- [x] Centrifugo coordination via `currentStreamingMessageIdAtom`
- [x] ChatContainer integration with streaming/thinking indicators
- [x] Build passes, linting passes

### PR1.7 Deliverables (All Working ✅)

- [x] Centrifugo integration for real-time file updates
- [x] Pending changes UI indicators (yellow dots)
- [x] Commit/Discard functionality for revision management
- [x] Files properly display in Explorer for revision 0

### PR1.65 Deliverables (All Working ✅)

- [x] Three-panel layout: Chat LEFT → Explorer MIDDLE → Code Editor RIGHT
- [x] Code editor panel with Monaco syntax highlighting
- [x] Source/Rendered tabs for file viewing
- [x] Loading states and visual feedback improvements
- [x] Landing page polish with upload options
- [x] ArtifactHubSearchModal integration
- [x] Consistent styling matching main workspace path

### PR1.61 Deliverables (All Working ✅)

- [x] Request-level body parameter in `sendMessage()` calls
- [x] `getChatBody()` helper with `useCallback` memoization
- [x] Fixed streaming on first message from entry point
- [x] Fixed tool execution (no more XML hallucination)
- [x] Correct answers returned (18.1.13 not 16.2.7)

### PR1.6 Deliverables (All Working ✅)

- [x] `lib/ai/prompts.ts` - Simplified system prompt (fixes tool execution)
- [x] `app/test-ai-chat/page.tsx` - Landing page with workspace creation
- [x] `app/test-ai-chat/[workspaceId]/page.tsx` - Server component data fetching
- [x] `app/test-ai-chat/[workspaceId]/client.tsx` - Client with FileBrowser + persistence
- [x] `lib/workspace/actions/create-ai-sdk-chat-message.ts` - User message persistence
- [x] `lib/workspace/actions/update-chat-message-response.ts` - AI response persistence

### PR1.5 Deliverables (All Working ✅)

- [x] `pkg/api/server.go` - Go HTTP server on port 8080
- [x] `pkg/api/errors.go` - Standardized error utilities
- [x] `pkg/api/handlers/editor.go` - textEditor endpoint
- [x] `pkg/api/handlers/versions.go` - Version lookup endpoints
- [x] `lib/ai/tools/utils.ts` - `callGoEndpoint` HTTP utility
- [x] All 4 tools: getChartContext, textEditor, latestSubchartVersion, latestKubernetesVersion
- [x] 22 integration tests passing

### PR1 Deliverables (All Working ✅)

- [x] `/api/chat/route.ts` - API route using `streamText`
- [x] `lib/ai/provider.ts` - Provider factory with direct API priority
- [x] `lib/ai/models.ts` - Model definitions (Claude Sonnet 4 default)
- [x] `components/chat/AIChat.tsx` - Chat UI with `useChat` hook
- [x] 61 unit tests passing

---

## What Needs to be Built

### PR2: Validation Agent (Days 8-10)

| Task | Status | Priority |
|------|--------|----------|
| Create `pkg/validation/pipeline.go` | 🔲 | Must |
| Create `pkg/validation/helm.go` | 🔲 | Must |
| Create `pkg/validation/kubescore.go` | 🔲 | Must |
| Create `pkg/validation/parser.go` | 🔲 | Must |
| Create `pkg/validation/types.go` | 🔲 | Must |
| Create `pkg/api/handlers/validate.go` | 🔲 | Must |
| Create `lib/ai/tools/validateChart.ts` | 🔲 | Must |
| Create `components/chat/ValidationResults.tsx` | 🔲 | Must |
| Security: Path traversal prevention | 🔲 | Must |
| Security: Command injection prevention | 🔲 | Must |

### Post-PR2.0 Cleanup (After Stable Period)

| Task | Status | Priority |
|------|--------|----------|
| Remove `/test-ai-chat` directory | 🔲 | Should |
| Remove `components/chat/AIChat.tsx` | 🔲 | Should |
| Remove `components/chat/AIMessageList.tsx` | 🔲 | Should |
| Remove feature flag (make AI SDK only path) | 🔲 | Should |

### PR3 Parity (Option A) - Upcoming Work
| Task | Status | Priority |
|------|--------|----------|
| DB migration: add `workspace_plan.buffered_tool_calls` JSONB + index | 🔲 | Must |
| Phase 1: Page reload guard + rule-based followups | 🔲 | Must |
| Phase 2: Intent classify endpoint + rollback field population | 🔲 | Must |
| Phase 3: Plan workflow (buffer tools, create plan, proceed/ignore) | 🔲 | Must |
| Phase 4: K8s conversion bridge tool + endpoint | 🔲 | Should |
| Parity stance: Option A (same UI/flow, AI prose may differ) | ✅ | Decision |

---

## Success Criteria Checklist

### PR2.0 ✅ Complete
- [x] Feature flag toggles legacy vs AI SDK path
- [x] Legacy path unchanged when flag is false
- [x] Persona affects prompt (visible in request body)
- [x] Streaming/cancel/state flags behave per mapping
- [x] Partial response persists on cancel
- [x] Commit/Discard works as review in AI SDK mode
- [x] Centrifugo events update UI without conflicts
- [x] 33 unit tests pass for message mapper
- [x] Build passes, linting passes
- [x] No regressions in legacy path

### PR1.7 ✅ Complete
- [x] Centrifugo integration for real-time updates
- [x] Pending changes UI indicators
- [x] Commit/Discard functionality

### PR1-PR1.65 ✅ Complete
- [x] All previous PR deliverables working

### PR2 (Pending)
- [ ] "Validate my chart" triggers validation tool
- [ ] helm lint, helm template, kube-score all execute
- [ ] Results displayed with severity indicators
- [ ] AI explains issues in natural language
- [ ] Fix suggestions provided for each issue
- [ ] Path validation prevents traversal attacks

---

## Known Issues / Risks

### Resolved Issues (All PRs through PR2.0)

| Issue | Resolution |
|-------|------------|
| AI SDK v5 different from v4 docs | Discovered correct patterns (UIMessage, default transport) |
| Tool execution outputs XML text | Simplified system prompt - let AI SDK handle tool schemas |
| No streaming on first message | Request-level body in sendMessage() - PR1.61 |
| Double processing (Go + AI SDK) | ChatMessageIntent.NON_PLAN added - PR1.7 prereq |
| File create goes to wrong column | AddFileToChartPending() writes to content_pending |
| **React hooks warnings in TtlshModal** | **Refactored to use useRef for interval, proper dependency chains** |
| **Build cache error for admin/waitlist** | **Cleared .next cache** |

### Remaining Risks (PR2)

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| helm CLI not installed | Medium | High | Check at startup, clear error |
| kube-score CLI not installed | Medium | High | Check at startup, clear error |
| Path traversal attacks | Low | High | Comprehensive validation |
| Command injection | Low | Critical | Use exec.Command with args array |

---

## Documentation Status

| Document | Status | Location |
|----------|--------|----------|
| PR2.0 Completion Report | ✅ Complete | `docs/PR2.0_COMPLETION_REPORT.md` |
| PR2.0 Tech PRD | ✅ Complete | `PRDs/PR2.0_REINTEGRATION_Tech_PRD.md` |
| PR2.0 Implementation Plan | ✅ Complete | `PRDs/PR2.0_REINTEGRATION_IMPLEMENTATION_PLAN.md` |
| PR1.6 Completion Report | ✅ Complete | `docs/PR1.6_COMPLETION_REPORT.md` |
| PR1.5 Completion Report | ✅ Complete | `docs/PR1.5_COMPLETION_REPORT.md` |
| PR1 Completion Report | ✅ Complete | `docs/PR1_COMPLETION_REPORT.md` |
| All PRDs | ✅ Complete | `PRDs/*.md` |
| Memory Bank | ✅ Updated | `memory-bank/*` |

---

## Test Results

### PR2.0 Message Mapper Tests
```
PASS lib/chat/__tests__/messageMapper.test.ts
Test Suites: 1 passed, 1 total
Tests:       33 passed, 33 total
Time:        0.136 s
```

### PR1.5 Integration Tests
```
Test Suites: 1 passed, 1 total
Tests:       22 passed, 22 total
Time:        0.505 s
```

### Unit Tests (Combined)
```
Test Suites: 8 passed, 8 total
Tests:       94 passed, 94 total
```

### Build Status
```
✓ Compiled successfully
✓ Linting passed
✓ TypeScript type check passed
✓ Production build successful
```

---

## Version History

| Date | Change |
|------|--------|
| Dec 2, 2025 | Memory bank created with all PRDs analyzed |
| Dec 4, 2025 | PR1 complete, memory bank updated |
| Dec 4, 2025 | PR1.5 complete, all tests passing, memory bank updated |
| Dec 4, 2025 | PR1.6 complete, feature parity achieved, memory bank updated |
| Dec 5, 2025 | PR1.61 complete, body parameter hotfix applied |
| Dec 5, 2025 | PR1.65 complete, UI feature parity achieved |
| Dec 5, 2025 | PR1.7 prereq fixes complete |
| Dec 5, 2025 | PR1.7 complete, Centrifugo + Commit/Discard working |
| Dec 5, 2025 | **PR2.0 complete**, AI SDK reintegrated into main workspace path |
| Dec 6, 2025 | PR3.0 parity plan finalized (Option A, buffered plan workflow) |

---

*Update this document as tasks are completed. Last updated: Dec 6, 2025 (PR3.0 plan ready)*
