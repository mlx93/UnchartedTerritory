# Chartsmith Migration Progress

**Purpose**: Track what works, what's left to build, current status, and known issues.

---

## Overall Status

| Phase | Status | Notes |
|-------|--------|-------|
| Research & Analysis | ✅ Complete | Codebase analyzed, architecture understood |
| PRD Creation | ✅ Complete | All PRDs written and validated |
| Architecture Decisions | ✅ Complete | 8 key decisions documented |
| PR1 Implementation | 🔲 Not Started | Ready to begin |
| PR1.5 Implementation | 🔲 Not Started | Awaiting PR1 |
| PR2 Implementation | 🔲 Not Started | Awaiting PR1.5 |

---

## What Already Works (Current Chartsmith)

### Frontend Features
- [x] Chat interface with streaming responses
- [x] Message history display
- [x] Jotai state management
- [x] Centrifugo real-time updates
- [x] VS Code extension integration
- [x] OAuth authentication (Google)

### Backend Features
- [x] PostgreSQL LISTEN/NOTIFY queue processing
- [x] Anthropic SDK integration (Go)
- [x] 3 tools: text_editor, latest_subchart_version, latest_kubernetes_version
- [x] Chart creation and revision management
- [x] File system operations
- [x] ArtifactHub integration

### Infrastructure
- [x] Docker Compose development environment
- [x] Jest unit tests
- [x] Playwright E2E tests
- [x] Go unit tests

---

## What Needs to be Built

### PR1: AI SDK Foundation (Days 1-3)

| Task | Status | Priority |
|------|--------|----------|
| Install new dependencies | 🔲 | Must |
| Create provider factory (`lib/ai/provider.ts`) | 🔲 | Must |
| Create model configuration (`lib/ai/models.ts`) | 🔲 | Must |
| Create `/api/chat` route | 🔲 | Must |
| Create ProviderSelector component | 🔲 | Must |
| Migrate Chat component to useChat | 🔲 | Must |
| Update message rendering for parts | 🔲 | Must |
| Add environment variables | 🔲 | Must |
| Update tests | 🔲 | Must |
| Update ARCHITECTURE.md | 🔲 | Must |
| Delete orphaned `lib/llm/prompt-type.ts` | 🔲 | Should |

### PR1.5: Migration & Feature Parity (Days 3-4)

| Task | Status | Priority |
|------|--------|----------|
| Create shared LLM client (`lib/ai/llmClient.ts`) | 🔲 | Must |
| Create Go HTTP server (`pkg/api/server.go`) | 🔲 | Must |
| Create error response utilities (`pkg/api/errors.go`) | 🔲 | Must |
| Create createChart tool + endpoint | 🔲 | Must |
| Create getChartContext tool + endpoint | 🔲 | Nice |
| Create updateChart tool + endpoint | 🔲 | Nice |
| Create textEditor tool + endpoint | 🔲 | Must |
| Create latestSubchartVersion tool + endpoint | 🔲 | Must |
| Create latestKubernetesVersion tool + endpoint | 🔲 | Must |
| Migrate system prompts to Node | 🔲 | Must |
| Remove @anthropic-ai/sdk | 🔲 | Must |
| Create integration test | 🔲 | Must |
| Mark legacy files as deprecated | 🔲 | Should |

### PR2: Validation Agent (Days 4-6)

| Task | Status | Priority |
|------|--------|----------|
| Create validation pipeline (`pkg/validation/`) | 🔲 | Must |
| Implement helm lint integration | 🔲 | Must |
| Implement helm template integration | 🔲 | Must |
| Implement kube-score integration | 🔲 | Must |
| Create validateChart tool | 🔲 | Must |
| Create ValidationResults component | 🔲 | Must |
| Create LiveProviderSwitcher component | 🔲 | Must |
| Update Chat for live provider switching | 🔲 | Must |
| Add validation progress indicator | 🔲 | Should |
| Handle missing CLI tools gracefully | 🔲 | Must |

---

## Success Criteria Checklist

### PR1
- [ ] Chat functionality equivalent to current implementation
- [ ] Streaming works with no visible degradation
- [ ] Provider selector allows choosing between GPT-4o and Claude
- [ ] All existing tests pass (or updated appropriately)
- [ ] VS Code extension authentication works
- [ ] No console errors during normal operation

### PR1.5
- [ ] "Create a nginx chart" works end-to-end via AI SDK chat
- [ ] createChart tool invokes Go endpoint correctly
- [ ] Chart appears in database after creation
- [ ] latestSubchartVersion returns valid version data
- [ ] latestKubernetesVersion returns valid version data
- [ ] textEditor view/create/str_replace work correctly
- [ ] No @anthropic-ai/sdk in node_modules
- [ ] Integration tests pass
- [ ] Error responses follow standard format

### PR2
- [ ] "Validate my chart" triggers validation tool
- [ ] helm lint, helm template, kube-score all execute
- [ ] Results displayed with severity indicators
- [ ] AI explains issues in natural language
- [ ] Fix suggestions provided for each issue
- [ ] Live provider switching works without losing history
- [ ] All PR1/PR1.5 functionality still works

---

## Known Issues / Risks

### Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| SDK version incompatibility | Medium | High | Pin exact versions, test thoroughly |
| OpenRouter downtime | Low | Medium | Document fallback to direct OpenAI |
| Streaming behavior differences | Medium | Medium | Thorough testing of edge cases |
| State management migration bugs | Medium | High | Incremental migration, extensive testing |
| Go HTTP server startup issues | Medium | High | Start in goroutine, log errors |
| Tool schema incompatibility | Low | Medium | Validate against AI SDK docs |

### Known Codebase Issues

1. **Orphaned Code**: `lib/llm/prompt-type.ts` contains `promptType()` function that is never called - safe to delete

2. **No `insert` Command**: The text_editor tool only supports `view`, `create`, `str_replace` (NOT `insert`) - PRD verified this

3. **Hardcoded K8s Version**: `latestKubernetesVersion` returns hardcoded values intentionally for stability

---

## Documentation Status

| Document | Status | Location |
|----------|--------|----------|
| Main PRD (PR1 Product) | ✅ Complete | `PRDs/PR1_Product_PRD.md` |
| Tech PRD (PR1 Tech) | ✅ Complete | `PRDs/PR1_Tech_PRD.md` |
| PR1.5 Product PRD | ✅ Complete | `PRDs/PR1.5_Product_PRD.md` |
| PR1.5 Tech PRD | ✅ Complete | `PRDs/PR1.5_Tech_PRD.md` |
| PR1.5 Plan | ✅ Complete | `PRDs/PR1.5_PLAN.md` |
| PR2 Product PRD | ✅ Complete | `PRDs/PR2_Product_PRD.md` |
| PR2 Tech PRD | ✅ Complete | `PRDs/PR2_Tech_PRD.md` |
| Architecture Decisions | ✅ Complete | `PRDs/CHARTSMITH_ARCHITECTURE_DECISIONS.md` |
| Current State Analysis | ✅ Complete | `ClaudeResearch/CURRENT_STATE_ANALYSIS.md` |
| Memory Bank | ✅ Complete | `memory-bank/*` |

---

## Next Actions

1. **Immediate**: Set up development environment
   - Clone chartsmith repo
   - Set up Docker Compose (PostgreSQL + Centrifugo)
   - Configure environment variables

2. **PR1 Start**: Create feature branch and install dependencies
   - `npm install ai @ai-sdk/react @openrouter/ai-sdk-provider`
   - Create `lib/ai/provider.ts`

3. **First Milestone**: Basic streaming working
   - Create `/api/chat/route.ts`
   - Test with simple message
   - Verify OpenRouter connectivity

---

## Version History

| Date | Change |
|------|--------|
| Dec 2, 2025 | Memory bank created with all PRDs analyzed |

---

*Update this document as tasks are completed.*

