# Implementation Guide

This document lists all files needed to implement PR1 and PR2, organized by priority.

---

## Quick Reference: What You Need

### For Implementation

| Priority | Files | Purpose |
|----------|-------|---------|
| **Primary** | PRDs (4 files) | Implementation specs |
| **Reference** | ClaudeResearch (4 files) | Codebase understanding |
| **Architecture** | Decision docs (2 files) | Key decisions |

### The Chartsmith Codebase

You'll work directly in `chartsmith/` - the PRDs tell you exactly what to create/modify.

---

## Primary Implementation Documents (Must Read)

These contain everything needed to implement:

### PR1 (Days 1-3)

| File | Content |
|------|---------|
| `PRDs/PR1_Product_PRD.md` | User stories, acceptance criteria, UI specs |
| `PRDs/PR1_Tech_PRD.md` | Technical spec, file structure, code patterns |

### PR2 (Days 4-6)

| File | Content |
|------|---------|
| `PRDs/PR2_Product_PRD.md` | Validation agent user stories, UI specs |
| `PRDs/PR2_Tech_PRD.md` | Go validation pipeline, tool definition, components |
| `PRDs/PR2_SUPPLEMENTAL_DETAILS.md` | Component pseudocode, rendering logic |

---

## Reference Documents (Consult As Needed)

These provide codebase context when PRDs reference existing patterns:

### ClaudeResearch Folder

| File | When to Reference |
|------|-------------------|
| `ClaudeResearch/CURRENT_STATE_ANALYSIS.md` | Understanding existing architecture |
| `ClaudeResearch/ARCHITECTURE_DECISIONS.md` | ADRs and rationale |
| `ClaudeResearch/VERCEL_AI_SDK_REFERENCE.md` | AI SDK patterns and APIs |
| `ClaudeResearch/VALIDATION_TOOLS_REFERENCE.md` | helm/kube-score CLI reference |

### Architecture Decisions

| File | When to Reference |
|------|-------------------|
| `docs/PR2_HTTP_ARCHITECTURE_DECISION.md` | Why HTTP for tool calling |
| `PRDs/PR2_ARCHITECTURE_DECISION_REQUIRED.md` | Full options analysis |

---

## Files You Can Skip

These were used during research but aren't needed for implementation:

| File | Reason to Skip |
|------|----------------|
| `PRDs/PR2_GAPS_ANALYSIS.md` | Research artifact - gaps already resolved in PRDs |
| `docs/research/*.md` | Research notes - findings incorporated into PRDs |
| `Replicated_Chartsmith.md` | Requirements doc - already validated against |
| `Uncharted_Territory_Challenge.md` | Challenge spec - already validated against |

---

## Chartsmith Codebase: Key Locations

When implementing, you'll work in these areas:

### PR1: Frontend Changes

```
chartsmith/chartsmith-app/
├── app/api/chat/route.ts           # CREATE - New chat API route
├── lib/ai/
│   ├── provider.ts                 # CREATE - Provider factory
│   ├── models.ts                   # CREATE - Model definitions
│   └── config.ts                   # CREATE - AI configuration
├── components/chat/
│   └── ProviderSelector.tsx        # CREATE - Model selector UI
├── components/ChatContainer.tsx    # MODIFY - Integrate useChat
├── components/ChatMessage.tsx      # MODIFY - Parts-based rendering
├── lib/llm/prompt-type.ts          # DELETE - Orphaned code
└── package.json                    # MODIFY - Add AI SDK packages
```

### PR2: Go Backend Changes

```
chartsmith/
├── cmd/run.go                      # MODIFY - Start HTTP server
├── pkg/api/
│   └── validate.go                 # CREATE - HTTP handler
├── pkg/validation/
│   ├── pipeline.go                 # CREATE - Orchestration
│   ├── helm.go                     # CREATE - helm lint/template
│   ├── kubescore.go                # CREATE - kube-score
│   ├── parser.go                   # CREATE - Output parsing
│   └── types.go                    # CREATE - Type definitions
```

### PR2: Frontend Tool Integration

```
chartsmith/chartsmith-app/
├── lib/ai/tools/
│   └── validateChart.ts            # CREATE - Tool definition
├── components/chat/
│   ├── ValidationResults.tsx       # CREATE - Results display
│   └── LiveProviderSwitcher.tsx    # CREATE - Live switching
└── app/api/chat/route.ts           # MODIFY - Register tool
```

---

## Existing Code to Study (When PRDs Reference Patterns)

### Go Patterns

| Pattern | Location | Study For |
|---------|----------|-----------|
| PostgreSQL listener | `pkg/listener/start.go` | Understanding existing architecture |
| Tool execution | `pkg/llm/execute-action.go:510-673` | Tool loop pattern |
| Centrifugo events | `pkg/realtime/centrifugo.go` | Event publishing |
| Helm execution | `helm-utils/render-exec.go` | Temp dir + exec pattern |

### Frontend Patterns

| Pattern | Location | Study For |
|---------|----------|-----------|
| Jotai atoms | `atoms/workspace.ts` | State management |
| Centrifugo hook | `hooks/useCentrifugo.ts` | Real-time events |
| Chat components | `components/ChatContainer.tsx` | Existing chat UI |

---

## Implementation Order

### PR1 Sequence

1. **Day 1 Morning**: Install deps, create provider factory, add env vars
2. **Day 1 Afternoon**: Create `/api/chat` route with streamText
3. **Day 2 Morning**: Create ProviderSelector, migrate Chat to useChat
4. **Day 2 Afternoon**: Update message rendering for parts
5. **Day 3**: Testing, docs, cleanup

### PR2 Sequence

1. **Day 4 Morning**: Create `pkg/validation/` with types and helm.go
2. **Day 4 Afternoon**: Add kubescore.go, pipeline.go, parser.go
3. **Day 5 Morning**: Create `pkg/api/validate.go`, start HTTP server
4. **Day 5 Afternoon**: Create validateChart tool, ValidationResults component
5. **Day 6**: LiveProviderSwitcher, testing, docs

---

## Environment Setup

### Required Environment Variables

```env
# PR1 - Frontend
OPENROUTER_API_KEY=sk-or-v1-xxxxx
DEFAULT_AI_PROVIDER=openai
DEFAULT_AI_MODEL=openai/gpt-4o

# PR2 - Go Backend Communication
GO_BACKEND_URL=http://localhost:8080
```

### Required CLI Tools (PR2)

```bash
# Verify these are installed for validation pipeline
helm version    # v3.x required
kube-score version  # v1.16+ recommended (optional - graceful degradation)
```

---

## Summary: Minimal File Set

**To implement PR1+PR2, you need:**

1. ✅ `PRDs/PR1_Product_PRD.md`
2. ✅ `PRDs/PR1_Tech_PRD.md`
3. ✅ `PRDs/PR2_Product_PRD.md`
4. ✅ `PRDs/PR2_Tech_PRD.md`
5. ✅ `PRDs/PR2_SUPPLEMENTAL_DETAILS.md`

**Helpful context (consult as needed):**

6. `ClaudeResearch/CURRENT_STATE_ANALYSIS.md`
7. `ClaudeResearch/ARCHITECTURE_DECISIONS.md`
8. `docs/PR2_HTTP_ARCHITECTURE_DECISION.md`

**Everything else is research artifacts** - already incorporated into the PRDs.
