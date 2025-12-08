# Chartsmith Brainlift

**AI SDK Migration & Validation Agent Project**

*Uncharted Territory Challenge — December 2025*

---

## 1. Purpose

> *A clear mission statement defining what problem you're solving and why it matters. This keeps exploration focused and prevents scope creep.*

**Challenge Alignment:** Brownfield proof via the Chartsmith fork, working in new territory across Go backend + Next.js/TypeScript frontend, delivering non-trivial complexity (plan workflow interception/buffering and the validation agent pipeline), and producing the required Uncharted outputs (comprehensive docs, tests, and demo coverage).

### The Problem

Chartsmith, an AI-powered Helm chart creation tool by Replicated, uses a custom **@anthropic-ai/sdk** integration with manual streaming implementation, tight provider coupling, and significant maintenance overhead. The architecture requires:

- Custom chat UI components with manual message handling
- Direct Anthropic SDK calls with no provider flexibility
- Complex state management across frontend and backend
- No AI-powered validation for generated Helm charts

### Why It Matters

This project demonstrates three critical engineering capabilities:

- **Brownfield Mastery:** Understanding and extending an unfamiliar codebase (Go backend + TypeScript frontend) without breaking existing functionality
- **SDK Migration Expertise:** Modernizing from custom AI integration to standardized Vercel AI SDK patterns while maintaining feature parity
- **Multi-Provider Architecture:** Enabling flexibility to switch between Claude, GPT-4, Gemini without code changes

### Scope Boundaries

**In Scope (Implemented through PR4):**
- Chat UI migration via adapter pattern (`useAISDKChatAdapter.ts`)
- Provider abstraction with Anthropic/OpenAI/OpenRouter support
- 6 AI SDK tools: getChartContext, textEditor, latestSubchartVersion, latestKubernetesVersion, convertK8sToHelm, validateChart
- Go HTTP server with 11 endpoints (port 8080)
- Plan workflow with tool buffering
- Intent classification via Groq
- Feature flag for safe rollout (`NEXT_PUBLIC_USE_AI_SDK_CHAT`)
- Validation agent with helm lint → helm template → kube-score pipeline
- Live provider switching mid-conversation (`LiveProviderSwitcher` component)

**Out of Scope:** VS Code extension changes, authentication modifications, new database tables (only added `buffered_tool_calls` column to existing `workspace_plan` table)

---

## 2. Research Methodology

> *How we systematically explored the codebase and made evidence-based decisions. This documents the tools, prompts, and processes that enabled deep understanding.*

### Tools Used

| Tool | Purpose | Key Output |
|------|---------|------------|
| **CodeLayer** | Codebase research with AI assistance | PR1/PR2 codebase analysis with file:line references |
| **Claude Code** | Agent-based research and implementation | Sub-agent architecture, completion reports, iterative implementation |
| **Custom Research Commands** | `/research_codebase` slash command | Structured research documents with YAML frontmatter |

### CodeLayer Research Process

**Setup** (`CodeLayerPrompts/CODELAYER_SETUP.md`):
1. Clone target repository
2. Create `docs/prds/` and `docs/research/` directories
3. Copy all PRD files into repository for CodeLayer access
4. Launch CodeLayer tool

**Research Prompt Template** (discovered pattern):
```
/research_codebase

Research the [codebase name] to answer these questions for [feature/project]:

## Questions to Answer

[Critical Questions]
1. [Targeted technical question with file hints]
2. [Integration mechanism question]
3. [Existing pattern inventory question]

[Medium Questions]
4. [Configuration/setup question]
5. [Data flow question]

## Context

[1-2 paragraphs describing feature, technology stack, migration goals]

## Reference Documents

Read these files in `docs/prds/` for full specifications:
- `[PRD_FILE_1.md]` - [Purpose]
- `[PRD_FILE_2.md]` - [Purpose]

## Output

Create/update:
- `docs/research/[OUTPUT_FILE.md]` with:
  - Answers to each question with `file:line` references
  - Architecture diagram (text-based)
  - Files requiring modification
  - Gaps/risks identified
```

**Research Output Quality Metrics**:
- PR1 Analysis: 571 lines, 50+ unique file references, 100+ line citations
- PR2 Findings: 461 lines, 30+ unique file references, 80+ line citations
- Every claim backed by specific `file:line` reference

### Research Document Structure

Every research document followed consistent YAML frontmatter:

```yaml
---
date: [ISO timestamp]
researcher: [Claude Code | mylessjs]
git_commit: [specific commit hash]
branch: main
repository: UnchartedTerritory
topic: "[descriptive title]"
tags: [research, codebase, specific-topics]
status: complete
last_updated: [date]
last_updated_by: [researcher]
---
```

This enabled traceability - every finding traced to exact codebase state at specific git commits.

### Research Progression Phases

| Phase | Documents | Key Output |
|-------|-----------|------------|
| **1. Initial Validation** | `2025-12-01-prd-validation-and-architecture-analysis.md` | Validated 5/5 PR1 assumptions accurate |
| **2. False Assumption Discovery** | `2025-12-02-PRD-FALSE-ASSUMPTIONS-ANALYSIS.md` | Identified 15 critical false assumptions |
| **3. Architecture Decision** | `PR2_ARCHITECTURE_DECISION_REQUIRED.md` | Evaluated 3 options, selected HTTP endpoint |
| **4. Implementation Research** | `2025-12-02-FULL-MIGRATION-OPTION-A.md` (1,319 lines) | Complete migration blueprint |
| **5. Gap Analysis** | `2025-12-04-PR1.6-PRD-GAPS-ANALYSIS.md` | 22 feature gaps identified |
| **6. Streaming Deep Dive** | `2025-12-04-PR1.6-AI-SDK-STREAMING-RESEARCH.md` | Root cause of tool execution failures |

---

## 3. Experts

> *The voices, research, and real-world evidence that validate or challenge our assumptions. This ensures we're grounding our thinking in reality.*

### Primary Sources Consulted

| Source | What It Validated / Challenged |
|--------|-------------------------------|
| **Vercel AI SDK v5 Documentation** | Confirmed `streamText` + `useChat` pattern. **Challenged:** v4 patterns in docs were outdated; TypeScript type definitions became authoritative source. |
| **CodeLayer Codebase Analysis** | Revealed PostgreSQL LISTEN/NOTIFY pattern for workspace events and Centrifugo WebSocket streaming. Discovered Go DOES make LLM calls via Anthropic SDK that needed migration. |
| **PR3.0 Implementation Plan** | Defined 4-phase approach for full feature parity. Validated tool buffering pattern for plan workflow integration. |
| **PR4 Tech PRD** | Specified validation pipeline: helm lint → helm template → kube-score with sequential execution (not parallel) and non-fatal kube-score failures. |
| **PR2.0 Completion Report** | Proved adapter pattern achieves feature parity in 8 hours vs 113+ hour rebuild estimate (75% effort reduction). |

### Key Research Findings (Final State)

**Finding 1: Original Architecture Had Go Making LLM Calls**
The Go backend DID make direct LLM calls via `anthropic-sdk-go`. This was the core of what we migrated. After migration, all LLM calls moved to Node/AI SDK; Go became pure application logic with HTTP endpoints.

**Finding 2: 6 AI SDK Tools Implemented (PR4 Complete)**
| Tool | Implementation | Purpose |
|------|----------------|---------|
| `getChartContext` | Go HTTP endpoint | Load workspace data (moved from TS due to `pg` bundling issues) |
| `textEditor` | Go HTTP endpoint | View/create/edit files (view=immediate, create/str_replace=buffered) |
| `latestSubchartVersion` | Go HTTP endpoint | ArtifactHub lookup |
| `latestKubernetesVersion` | Go HTTP endpoint | K8s version info |
| `convertK8sToHelm` | Go HTTP bridge | Triggers 6-step K8s→Helm conversion pipeline |
| `validateChart` | Go HTTP endpoint | Helm lint → template → kube-score validation pipeline |

**Finding 3: Workspace Creation Happens Before Chat**
Workspace creation uses the homepage TypeScript flow (`createWorkspaceFromPromptAction()`) BEFORE AI SDK chat begins. This eliminated the need for `createChart`, `updateChart`, `createEmptyWorkspace` tools.

**Finding 4: Tool Buffering Enables Plan Workflow**
PR3.0 discovered that `textEditor` `create`/`str_replace` calls must be buffered (not executed immediately) to support the plan approval workflow. The `toolInterceptor.ts` intercepts destructive calls, `bufferedTools.ts` returns success without executing, and `onFinish` calls Go to create a plan from buffered calls.

**Finding 5: Intent Classification Uses Groq (7 Intents)**
Go endpoint `/api/intent/classify` uses Groq's `llama-3.3-70b-versatile` for fast classification. Returns 7 boolean flags: `IsOffTopic`, `IsPlan`, `IsConversational`, `IsChartDeveloper`, `IsChartOperator`, `IsProceed`, `IsRender`. Note: `IsRollback` mentioned in early PRDs does NOT exist in the actual code.

**Finding 6: HTTP Endpoints for Synchronous Tool Execution**
After evaluating 3 options (HTTP, PostgreSQL queue, CLI subprocess), we chose HTTP endpoints for tool communication because AI SDK tools require synchronous responses. PostgreSQL queue pattern is async (unsuitable). Go HTTP server runs on port 8080 alongside existing PostgreSQL listeners.

**Finding 7: 11 Go HTTP Endpoints (Final Count)**
```
POST /api/tools/editor                    → handlers.TextEditor
POST /api/tools/versions/subchart         → handlers.GetSubchartVersion
POST /api/tools/versions/kubernetes       → handlers.GetKubernetesVersion
POST /api/tools/context                   → handlers.GetChartContext
POST /api/intent/classify                 → handlers.ClassifyIntent
POST /api/plan/create-from-tools          → handlers.CreatePlanFromToolCalls
POST /api/plan/publish-update             → handlers.PublishPlanUpdate
POST /api/plan/update-action-file-status  → handlers.UpdateActionFileStatus
POST /api/conversion/start                → handlers.StartConversion
POST /api/validate                        → handlers.ValidateChart
GET  /health                              → health check
```

---

## 4. SpikyPOVs

> *The proven cases where conventional wisdom turned out to be wrong. This builds a foundation of counter-consensus thinking that's backed by evidence.*

| Conventional Thinking | What We Actually Did | Evidence |
|-----------------------|----------------------|----------|
| "Rebuild all 27+ UI features for AI SDK path" | **Adapter pattern preserved all features** with 75% less effort (8 hours vs 113+ estimated) | `useAISDKChatAdapter.ts` (319 lines) converts AI SDK messages to legacy format |
| "Verbose system prompts help AI understand tools better" | **Simplified 92-line prompt to 20 lines;** AI SDK automatically passes tool schemas | Verbose prompts caused tool hallucination (AI output XML as text instead of calling tools) |
| "Execute file changes immediately when AI calls tool" | **Buffer destructive tool calls;** execute on user approval via plan workflow | `toolInterceptor.ts` + `bufferedTools.ts` intercept `create`/`str_replace`, Go creates plan |
| "Provider locked after first message (PR1 design)" | **PR4 enables live provider switching mid-conversation** without losing history | `useAISDKChatAdapter.ts` state management + `LiveProviderSwitcher` component |
| "Use PostgreSQL queue for tool communication (existing pattern)" | **HTTP endpoints for synchronous tool execution** - queue is async, tools need immediate results | Go HTTP server on port 8080 with 11 endpoints |
| "Need complex intent routing in AI SDK" | **Reuse existing Go/Groq intent classification;** call before AI SDK processes | `/api/intent/classify` returns 7 intents; TypeScript routes based on flags |
| "Migrate all tools including utility functions" | **6 tools total, not all-inclusive:** Workspace creation uses homepage flow, not tool | `createChart`, `updateChart`, `createEmptyWorkspace` handled by existing TypeScript, not AI |
| "K8s conversion needs full TypeScript port" | **HTTP bridge to existing Go pipeline;** 6-step conversion is battle-tested | `convertK8sToHelm` tool calls Go `/api/conversion/start`, reuses existing workers |

### Evidence for Key Decisions

#### Why Adapter Pattern Over Full Rewrite?

PR2.0 Completion Report documents: "Rather than rebuilding 27+ features to achieve parity (estimated 113-163 hours), we adapted the working AI SDK chat endpoint to work with the existing, feature-complete main path UI (estimated 30-40 hours, actual ~8 hours)."

The adapter (`useAISDKChatAdapter.ts`) converts AI SDK `UIMessage` format to legacy `Message` format:
- `parts[].text` (user) → `prompt`
- `parts[].text` (assistant) → `response`
- `status === 'streaming'` → `isComplete: false`

#### Why Tool Buffering Instead of Immediate Execution?

PR3.0 discovered that the legacy Go worker path had a mature plan approval workflow that users expected. Direct file changes would bypass this UX. The solution:

1. `textEditor` `view` command → Execute immediately (read-only, safe)
2. `textEditor` `create`/`str_replace` → Buffer in `toolInterceptor.ts`, return `{ success: true, buffered: true }`
3. `onFinish` callback → Call Go `/api/plan/create-from-tools` with buffered calls
4. `PlanChatMessage` UI → User sees Proceed/Ignore buttons
5. Proceed → Execute buffered calls via `/api/tools/editor`

**Key insight:** Content is buffered during streaming, executed at proceed time. Legacy path regenerates via LLM at proceed time. Same UX, different transport.

#### Why HTTP Endpoints Instead of PostgreSQL Queue?

Research evaluated 3 options for tool↔Go communication:

| Aspect | PostgreSQL Queue | HTTP | CLI Subprocess |
|--------|------------------|------|----------------|
| Latency | Higher (queue + notify + consume) | Lower (direct request/response) | Medium |
| AI SDK compatibility | ❌ Async - tools need sync responses | ✅ Sync - returns immediately | ✅ Sync |
| Error propagation | Hard | Natural HTTP status codes | Exit codes |
| Debugging | Complex | Simple (curl testable) | Medium |

**Decision:** HTTP endpoints because AI SDK `tool.execute()` must return results synchronously for the LLM to continue reasoning.

---

## 5. Knowledge Tree

> *The web of background context and considerations needed to understand the full picture. This helps identify blind spots and connections.*

### Final System Architecture (After PR4)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FINAL ARCHITECTURE (PR4 Complete)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────┐     useChat Hook    ┌───────────────────┐            │
│  │   Chat UI         │ ─────────────────── │  /api/chat        │ ─── Multi-Provider
│  │  (AI SDK-based)   │ ◄───────────────────│   (streamText)    │     (Anthropic/
│  │  @ai-sdk/react    │   Data Stream       │   + 6 AI SDK      │      OpenAI/
│  │                   │                     │     tools         │      OpenRouter)
│  │  + Feature Flag   │                     │   + bufferedTools │            │
│  │  + Provider       │                     │                   │            │
│  │    Switcher       │                     │                   │            │
│  └───────────────────┘                     └─────────┬─────────┘            │
│           │                                          │ HTTP (tool calls)     │
│           │ Live switching                           │                       │
│           ▼                                          │                       │
│  ┌───────────────────┐                               │                       │
│  │ LiveProviderSwitcher                              │                       │
│  │ (dropdown in chat)│                               │                       │
│  └───────────────────┘                               │                       │
│                                                      │                       │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │ ─ ─ ─ ─ ─ ─ ─ ─ ─    │
│  GO BACKEND (HTTP Server :8080 + PostgreSQL queue)   │                       │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │
│                                                      │                       │
│  ┌───────────────────┐              ┌────────────────▼───────────────────┐  │
│  │ Workspace APIs    │  pg_notify   │  Go HTTP Server (:8080)            │  │
│  │ (homepage flow)   │ ───────────▶ │  POST /api/tools/editor            │  │
│  └───────────────────┘              │  POST /api/tools/context           │  │
│                                     │  POST /api/tools/versions/*        │  │
│                                     │  POST /api/intent/classify         │  │
│                                     │  POST /api/plan/* (3 endpoints)    │  │
│                                     │  POST /api/conversion/start        │  │
│                                     │  POST /api/validate ◄── Validation │  │
│                                     │  GET  /health          Pipeline    │  │
│                                     └────────────────────────────────────┘  │
│                                                      │                       │
│                                     ┌────────────────▼───────────────────┐  │
│                                     │  pkg/validation/                   │  │
│                                     │  ┌─────────┐ ┌─────────┐ ┌───────┐ │  │
│                                     │  │helm lint│→│helm     │→│kube-  │ │  │
│                                     │  │         │ │template │ │score  │ │  │
│                                     │  └─────────┘ └─────────┘ └───────┘ │  │
│                                     └────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Transformations:**
- **Before:** Go made LLM calls via `anthropic-sdk-go`, all orchestration in Go workers
- **After:** Node/AI SDK makes all LLM calls, Go provides HTTP endpoints for tool execution

### AI SDK Tools (6 Implemented - PR4 Complete)

| Tool | Type | Go Endpoint | Behavior |
|------|------|-------------|----------|
| `getChartContext` | Immediate | `/api/tools/context` | Returns workspace files and metadata |
| `textEditor` | Buffered/Immediate | `/api/tools/editor` | `view`=immediate, `create`/`str_replace`=buffered |
| `latestSubchartVersion` | Immediate | `/api/tools/versions/subchart` | ArtifactHub lookup |
| `latestKubernetesVersion` | Immediate | `/api/tools/versions/kubernetes` | K8s version info |
| `convertK8sToHelm` | Bridge | `/api/conversion/start` | Triggers 6-step Go conversion pipeline |
| `validateChart` | Immediate | `/api/validate` | Helm lint → template → kube-score validation |

### PR Structure & Dependencies (Final)

| PR | Status | Key Deliverables |
|----|--------|------------------|
| **PR1** | ✅ Complete | /api/chat endpoint, useChat integration, provider factory, OpenRouter setup |
| **PR1.5** | ✅ Complete | 4 initial tools, Go HTTP server (port 8080), system prompts, remove `@anthropic-ai/sdk` |
| **PR1.6** | ✅ Complete | Test path workspace creation, file explorer, chat persistence, CSS fixes |
| **PR2.0** | ✅ Complete | Adapter pattern, message mapper, feature flag toggle, Centrifugo coordination |
| **PR3.0** | ✅ Complete | Intent classification, plan workflow, tool buffering, K8s conversion bridge, 5th tool |
| **PR4** | ✅ Complete | Chart Validation Agent (6th tool), Live Provider Switching, ValidationResults UI |

### PR4 Implementation Details

**Feature 1: Chart Validation Agent**

The validation pipeline is the primary showcase of Go development work:

| Component | Files Created | Purpose |
|-----------|---------------|---------|
| **pkg/validation/types.go** | Types | `ValidationRequest`, `ValidationResult`, `ValidationIssue` structures |
| **pkg/validation/helm.go** | Go | `runHelmLint()`, `runHelmTemplate()` with output parsing |
| **pkg/validation/kubescore.go** | Go | `runKubeScore()` with JSON parsing and severity mapping |
| **pkg/validation/pipeline.go** | Go | `RunValidation()` orchestrates 3-stage sequential pipeline |
| **pkg/api/handlers/validate.go** | Go | HTTP handler for `POST /api/validate` |
| **lib/ai/tools/validateChart.ts** | TypeScript | AI SDK tool factory |
| **atoms/validationAtoms.ts** | TypeScript | Jotai state management for validation results |
| **components/chat/ValidationResults.tsx** | React | UI component following `PlanChatMessage` pattern |

**Validation Pipeline Flow:**
```
helm lint → helm template → kube-score
   │             │              │
   ▼             ▼              ▼
Parse errors  Render YAML    Score manifests
& warnings    (for Stage 3)  (JSON output)
```

**Key Design Decisions:**
- Sequential execution (not parallel) for simpler error handling
- kube-score failure is non-fatal (partial results returned)
- Uses `workspace.ListCharts()` to get chart files (no path traversal risk)
- Grade mapping: kube-score Grade 1 → "critical", Grade 5 → "warning"
- `validateChart` is non-buffered (read-only, executes immediately)

**Feature 2: Live Provider Switching**

| Component | Changes | Purpose |
|-----------|---------|---------|
| **components/chat/LiveProviderSwitcher.tsx** | Created | Dropdown component in chat input area |
| **hooks/useAISDKChatAdapter.ts** | Extended | Added `selectedProvider`, `selectedModel`, `switchProvider()` state |
| **hooks/useLegacyChat.ts** | Extended | Interface compatibility for fallback path |
| **components/ChatContainer.tsx** | Modified | Integrated `LiveProviderSwitcher` next to role selector |

**Provider State Management:**
```typescript
// useAISDKChatAdapter additions
const [selectedProvider, setSelectedProvider] = useState<string>(DEFAULT_PROVIDER);
const [selectedModel, setSelectedModel] = useState<string>(DEFAULT_MODEL);

const switchProvider = useCallback((provider: string, model: string) => {
  setSelectedProvider(provider);
  setSelectedModel(model);
}, []);

// getChatBody() now uses dynamic provider/model
```

**Key Design Decisions:**
- Provider switching preserves full message history client-side
- No conversation interruption - next message uses new provider
- Available providers: Anthropic (Claude), OpenAI (GPT-4), OpenRouter

### Critical Context for Implementation

- **Feature Flag:** `NEXT_PUBLIC_USE_AI_SDK_CHAT=true` enables AI SDK path; `false` falls back to legacy Go worker
- **Instant Rollback:** Both paths use same database; toggle flag and redeploy to rollback
- **Go Learning Requirement:** PR4 validation pipeline is primary showcase for Go development work
- **Dual Project Compliance:** Must satisfy both Replicated Chartsmith spec and Uncharted Territory Challenge criteria

---

## 6. Agent Architecture & Prompting Strategies

> *The hierarchical agent system and prompt engineering patterns that enabled complex multi-phase implementation.*

### Master Agent Orchestrator Design

The project used a hierarchical multi-agent system (`agent-prompts/MASTER_AGENT_ORCHESTRATOR.md`):

```
┌─────────────────────────────────────┐
│     Master Agent Orchestrator       │
│  - Tracks dual requirements         │
│  - Manages PR dependency chain      │
│  - Spawns sub-agents               │
│  - Enforces cross-PR consistency    │
└──────────────┬──────────────────────┘
               │
    ┌──────────┼──────────┬──────────┬──────────┐
    ▼          ▼          ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ Setup  │ │  PR1   │ │ PR1.5  │ │ PR1.6  │ │  PR2   │
│ Agent  │ │Sub-Agt │ │Sub-Agt │ │Sub-Agt │ │Sub-Agt │
└────────┘ └────────┘ └────────┘ └────────┘ └────────┘
```

**Master Agent Responsibilities**:
1. **Context Maintenance**: Tracks both Replicated and Uncharted Territory requirements
2. **Dependency Management**: PR1 → PR1.5 → PR2 strict sequencing
3. **Document Tiering**: Essential (Tier 1) → High Value (Tier 2) → Contextual (Tier 3)
4. **Completion Verification**: Explicit checklists per PR
5. **Escalation Triggers**: Blocking issues, scope creep, time concerns

### Sub-Agent Prompt Structure

Every sub-agent prompt followed consistent structure:

```markdown
## Your Mission
Implement PR{X}: {Clear, specific goal}

## Primary Documents (READ FIRST - IN ORDER)
1. PRDs/PR{X}_Tech_PRD.md - Technical specification
2. PRDs/PR{X}_Product_PRD.md - Functional requirements
3. [CRITICAL docs with MUST READ tags]

## Previous PR Context (ALREADY COMPLETE)
[What exists, where to find it, how to build on it]

## Files to Create
[Exact file paths with descriptions]

## Files to Modify
[With discovery instructions where needed]

## Critical Architecture Points
[Key implementation details, patterns to follow]

## Implementation Priority
[Numbered list with time estimates]

## Success Criteria
[Explicit checklist items]

## Testing Strategy
[How to verify each component]

## DO NOT
[Explicit scope boundaries to prevent creep]

## Completion Requirements
[Required completion report format]
```

### Prompting Patterns That Worked

#### Pattern 1: Mission-First Framing
```
"Implement PR1.5: Tool Integration & Go HTTP Server for Chartsmith.
This is the **substantial Go work** demonstrating learning for
Uncharted Territory challenge."
```
Sets context immediately, reminds of broader goals.

#### Pattern 2: Document Priority Ordering
```
1. PRDs/PR1.5_PLAN.md - Task breakdown
2. PRDs/PR1.5_Tech_PRD.md - Technical spec
3. PRDs/PR1.5_Go_Handlers_Imp_Notes.md - CRITICAL - Exact function names
4. docs/research/2025-12-02-chartsmith-anthropic-tool-definitions.md - MUST READ
```
Guides agent through knowledge acquisition systematically.

#### Pattern 3: Embedded Code Snippets
```typescript
// System prompt replacement (exact code to use)
export const CHARTSMITH_TOOL_SYSTEM_PROMPT = `
You are a Helm chart expert assistant...
[Only behavioral guidelines, no tool documentation]
`;
```
Reduces ambiguity, ensures consistent implementation.

#### Pattern 4: Anti-Pattern Documentation
```markdown
## DO NOT
- Reimplement existing Go functions (call them, don't copy them)
- Modify the provider factory structure (llmClient.ts imports from it)
- Remove @anthropic-ai/sdk (already done in PR1)
- Add tool UI rendering (that's post-PR1.5 integration work)
```
Prevents scope creep and wasted effort.

#### Pattern 5: Discovery-Then-Implement
```
"Discover via CLAUDE.md - Find ChatContainer component"
"Router Discovery: Check go.mod for existing router"
```
Teaches pattern: understand before changing.

#### Pattern 6: Completion Report as Deliverable
Every sub-agent must create standardized completion report:
- Files summary table (Created/Modified/Deleted)
- Implementation notes (deviations, challenges)
- Code metrics (lines written, functions reused)
- Test results
- Known issues/TODOs for next PR
- Next PR readiness checklist

### Example: Generic vs. Brainlift-Powered Prompt

**Generic (Don't Do This):**
> "Implement the chat API route using Vercel AI SDK"

**Brainlift-Powered (Do This):**
> "Based on CodeLayer's finding that Chartsmith uses PostgreSQL LISTEN/NOTIFY for real-time events (pkg/notify/notify.go:45), and given that conventional wisdom suggests migrating all tools 1:1 but we've proven only 4 tools need porting, implement the /api/chat route using streamText with tool definitions for createChart, getChartContext, updateChart, and textEditor. The Go backend should receive NO LLM calls—only HTTP requests for chart CRUD operations. Reference the working pattern from 2025-12-02-chartsmith-anthropic-tool-definitions.md for exact tool schema format."

---

## 7. PR Evolution & Learning Journey

> *How the project evolved through research, implementation, and discovery. Documents the actual path taken vs. planned.*

### Timeline Overview

| PR | Days | Lines Added | Key Discovery |
|----|------|-------------|---------------|
| **PR1** | 1-3 | ~1,300 | AI SDK v5 API differs significantly from docs |
| **PR1 Addendum** | 3 | +200 | Direct provider support needed (not just OpenRouter) |
| **PR1.5** | 3-4 | ~1,295 | `getChartContext` must use Go HTTP (bundling issue) |
| **PR1.6** | 4-5 | ~600 | Verbose system prompt caused tool hallucination |
| **PR2.0** | 5 | ~910 | Adapter pattern 75% more efficient than rewrite |
| **PR3.0** | 6-7 | ~1,650 | Tool buffering enables plan workflow with streaming |
| **PR4** | 7-8 | ~1,200 | Validation UI follows `PlanChatMessage` pattern; provider switching via state |

### Critical Pivots

**Pivot 1: PR1 Scope Reduction**
- **Plan**: Migrate all Anthropic SDK usage to AI SDK
- **Discovery**: `lib/llm/prompt-type.ts` is orphaned code (zero imports)
- **Pivot**: Delete orphaned code, create NEW parallel system
- **Impact**: Simplified PR1 from migration to addition

**Pivot 2: Tool Communication Architecture**
- **Plan**: TypeScript-only `getChartContext` calling `getWorkspace()` directly
- **Discovery**: Next.js can't bundle `pg` module (requires `dns`, `net`, `tls`)
- **Pivot**: Move `getChartContext` to Go HTTP endpoint
- **Impact**: Added 4th Go endpoint (originally planned 3)

**Pivot 3: Tool Execution Failure**
- **Symptom**: AI outputting `<latestSubchartVersion>{"chartName":"postgresql"}</latestSubchartVersion>` as text
- **Discovery**: 92-line verbose system prompt competed with AI SDK's automatic schema
- **Pivot**: Simplified to 20-line minimal prompt
- **Impact**: Tools execute correctly, AI SDK handles schemas

**Pivot 4: Feature Parity Approach**
- **Plan**: Rebuild 27+ UI features for AI SDK path
- **Discovery**: Effort estimate 113-163 hours
- **Pivot**: Adapter pattern converting AI SDK messages to legacy format
- **Impact**: 75% effort reduction (~8 hours actual)

**Pivot 5: Plan Workflow Integration**
- **Plan**: Direct file changes via tools
- **Discovery**: Legacy path has mature approval workflow users expect
- **Pivot**: Tool buffering pattern - intercept create/str_replace, execute on approval
- **Impact**: Maintained existing UX while changing transport

### Major Challenges & Solutions

| Challenge | Root Cause | Solution | Files |
|-----------|------------|----------|-------|
| AI SDK v5 API mismatches | Documentation referenced v4 patterns | Used TypeScript definitions as source of truth | All `lib/ai/*.ts` |
| Node.js module bundling | `pg` requires Node-only modules | Moved `getChartContext` to Go HTTP | `pkg/api/handlers/context.go` |
| Tool execution hallucination | Verbose system prompt | Simplified prompt to 20 lines | `lib/ai/prompts.ts` |
| Feature parity scope | 27+ features, 113+ hours estimate | Adapter pattern | `hooks/useAISDKChatAdapter.ts` |
| Stale body values | AI SDK freezes `body` at init | Pass body in `sendMessage()` calls | Throughout test path |
| Plan workflow integration | Legacy has approval flow | Tool call buffering pattern | `lib/ai/tools/toolInterceptor.ts` |

### Code Metrics Summary

| Metric | Value |
|--------|-------|
| Total new production code | ~6,955 lines |
| Go code (new) | ~1,800 lines (includes pkg/validation/) |
| TypeScript code (new) | ~5,100 lines |
| Test coverage | 116 tests passing |
| Go HTTP endpoints | 11 total |
| AI SDK tools | 6 total |
| Frontend components (PR4) | ValidationResults, LiveProviderSwitcher |
| Research documents | 22 files, 15,000+ lines |

---

## 8. The 15 False Assumptions

> *Critical discoveries that changed our approach. Every assumption was disproven through codebase research.*

### Category 1: Architecture & Communication

| # | False Assumption | Reality | Impact |
|---|------------------|---------|--------|
| 1 | Next.js API Routes call LLM directly | Go workers process via PostgreSQL queue | Required new HTTP server |
| 2 | Go backend has HTTP endpoints | No HTTP server, only PostgreSQL listeners | Added `pkg/api/server.go` |
| 3 | Frontend uses Vercel AI SDK | No AI SDK installed, custom Jotai state | Full new installation needed |
| 4 | Streaming uses standard HTTP/SSE | Centrifugo WebSocket pub/sub | Coordination required |

### Category 2: Tool Calling & Validation

| # | False Assumption | Reality | Impact |
|---|------------------|---------|--------|
| 5 | Tools defined in AI SDK format | Anthropic-native JSON in Go | Translation required |
| 6 | Tool results flow through AI SDK | Custom message format via Centrifugo | Custom handling needed |
| 7 | No existing Helm CLI integration | `helm-utils/` already exists | Reuse existing code |
| 8 | No existing validation infrastructure | Dagger-based validation exists | Build on existing |

### Category 3: State Management & UI

| # | False Assumption | Reality | Impact |
|---|------------------|---------|--------|
| 9 | State management is React/Context | Custom Jotai atoms | Learn Jotai patterns |
| 10 | Chat component structure is simple | Complex multi-component architecture | Careful integration |
| 11 | Single /api/chat endpoint | Multiple workspace-scoped routes | Different approach |

### Category 4: Provider & Model

| # | False Assumption | Reality | Impact |
|---|------------------|---------|--------|
| 12 | OpenRouter is only provider option | Go uses Anthropic SDK directly | Multiple provider support |
| 13 | Model switching is frontend-only | Models hardcoded in Go backend | Go changes needed |

### Category 5: Testing & Code

| # | False Assumption | Reality | Impact |
|---|------------------|---------|--------|
| 14 | Test fixtures need creation | Comprehensive fixtures in 3 locations | Reuse existing |
| 15 | Frontend Anthropic SDK is heavy | ONE function only, rest orphaned | Simple deletion |

---

## 9. Key Files Reference

> *Critical files organized by function for quick navigation.*

### AI SDK Integration
- `chartsmith-app/app/api/chat/route.ts` - Main streaming endpoint with intent classification, buffered tools, onFinish plan creation
- `chartsmith-app/lib/ai/provider.ts` - Provider factory with priority chain (Anthropic → OpenAI → OpenRouter)
- `chartsmith-app/lib/ai/prompts.ts` - System prompts (simplified to 20 lines, persona support)
- `chartsmith-app/lib/ai/config.ts` - AI constants, defaults, model definitions

### Tool Implementations (6 Tools - PR4 Complete)
| TypeScript | Go Handler | Behavior |
|------------|------------|----------|
| `lib/ai/tools/getChartContext.ts` | `pkg/api/handlers/context.go` | Immediate |
| `lib/ai/tools/textEditor.ts` | `pkg/api/handlers/editor.go` | view=immediate, create/str_replace=buffered |
| `lib/ai/tools/latestSubchartVersion.ts` | `pkg/api/handlers/versions.go` | Immediate |
| `lib/ai/tools/latestKubernetesVersion.ts` | `pkg/api/handlers/versions.go` | Immediate |
| `lib/ai/tools/convertK8s.ts` | `pkg/api/handlers/conversion.go` | Bridge to Go pipeline |
| `lib/ai/tools/validateChart.ts` | `pkg/api/handlers/validate.go` | Immediate (validation pipeline) |

### Adapter & Integration (PR2.0/PR3.0)
- `hooks/useAISDKChatAdapter.ts` - Bridges AI SDK to legacy UI (319 lines)
- `hooks/useLegacyChat.ts` - Wrapper for Go worker fallback (92 lines)
- `lib/chat/messageMapper.ts` - UIMessage ↔ Message format conversion (283 lines, 35 tests)
- `lib/ai/tools/toolInterceptor.ts` - Tool call buffering infrastructure
- `lib/ai/tools/bufferedTools.ts` - Buffered tool wrappers (view=immediate, create/str_replace=buffered)

### Go Backend (11 Endpoints)
- `pkg/api/server.go` - HTTP server (port 8080), route registration
- `pkg/api/handlers/context.go` - GET chart context
- `pkg/api/handlers/editor.go` - textEditor operations (210 lines)
- `pkg/api/handlers/versions.go` - ArtifactHub and K8s version lookups
- `pkg/api/handlers/intent.go` - Groq-based intent classification (82 lines)
- `pkg/api/handlers/plan.go` - Plan endpoints (create-from-tools, publish-update, update-action-file-status)
- `pkg/api/handlers/conversion.go` - K8s→Helm conversion bridge (180 lines)
- `pkg/api/handlers/validate.go` - Chart validation pipeline (helm lint → template → kube-score)

### Plan Workflow (PR3.0)
- `lib/ai/plan.ts` - createPlanFromToolCalls client (55 lines)
- `lib/ai/intent.ts` - Intent classification client + routing (108 lines)
- `lib/workspace/actions/proceed-plan.ts` - Execute buffered tools on Proceed (168 lines)
- `components/PlanChatMessage.tsx` - Plan UI with Proceed/Ignore buttons

### Validation Pipeline (PR4)
- `pkg/validation/types.go` - ValidationRequest/Result types and issue structures
- `pkg/validation/helm.go` - helm lint and helm template execution with output parsing
- `pkg/validation/kubescore.go` - kube-score execution with JSON parsing and suggestions
- `pkg/validation/pipeline.go` - Three-stage validation orchestration using `workspace.ListCharts()`
- `lib/ai/tools/validateChart.ts` - AI SDK tool factory (non-buffered, immediate)
- `atoms/validationAtoms.ts` - Validation state management (Jotai)
- `components/chat/ValidationResults.tsx` - Validation UI following `PlanChatMessage` pattern

### Live Provider Switching (PR4)
- `components/chat/LiveProviderSwitcher.tsx` - Dropdown component in chat input area
- `hooks/useAISDKChatAdapter.ts` - Extended with `selectedProvider`, `selectedModel`, `switchProvider()`

### State Management
- `atoms/workspace.ts` - Jotai atoms for workspace, plans, conversions
- `atoms/validationAtoms.ts` - Jotai atoms for validation results (PR4)
- `hooks/useCentrifugo.ts` - Real-time updates with streaming coordination (skips streaming messages)

---

## 10. Using This Brainlift

> *Practical guidance for leveraging this research in prompts to sub-agents and implementation sessions.*

### Prompt Engineering Patterns

#### Use Expert Insights as Context

*Start prompts with:* "Based on CodeLayer's finding that Chartsmith uses PostgreSQL LISTEN/NOTIFY for real-time events..."

This grounds the LLM's response in specific expertise rather than general knowledge.

#### Lead with SpikyPOVs

*Frame questions with:* "Given that conventional wisdom suggests migrating all tools 1:1 but we've proven only 4 tools need porting..."

This pushes the LLM to think beyond consensus views.

#### Leverage the Knowledge Tree

*Include relevant background:* "Considering the three-layer architecture (Frontend/Go/AI) and that Go handles NO LLM calls..."

This helps the LLM make connections across domains.

#### Stay Focused with Purpose

*Reference scope:* "Focusing specifically on the /api/chat endpoint migration, NOT the validation pipeline..."

This keeps the LLM from wandering into general advice.

#### Reference False Assumptions

*Prevent known mistakes:* "Remembering that assumption #2 was false - Go has NO HTTP server in the original architecture..."

This prevents re-discovering known pitfalls.

### Research Command Pattern

When needing codebase research, use structured prompts:

```
/research_codebase

Research [specific area] to answer:
1. [Precise technical question]
2. [Integration question]
3. [Pattern inventory question]

Context: [What you know, what you need]

Output: docs/research/[date]-[topic].md with file:line references
```

### Completion Report Template

After implementing a PR, create:

```markdown
# PR{X} Completion Report

## Files Summary
| Action | File | Lines | Purpose |
|--------|------|-------|---------|
| Created | ... | ... | ... |
| Modified | ... | ... | ... |

## Implementation Notes
- What deviated from spec and why
- Challenges encountered
- Patterns discovered

## Code Metrics
- New lines: X
- Functions reused: Y
- Test coverage: Z%

## Known Issues
- [Issue with mitigation]

## Next PR Readiness
- [ ] Checklist item 1
- [ ] Checklist item 2
```

---

## 11. Lessons Learned

> *Meta-insights about the research and implementation process itself.*

### On Research

1. **TypeScript definitions > Documentation**: AI SDK v5 docs lagged behind actual API. Type definitions were authoritative source.

2. **Evidence-based claims only**: Every finding backed by `file:line` reference. Prevents assumptions from becoming "facts".

3. **Assumption validation first**: Validate PRD assumptions against codebase before implementing. Found 15 false assumptions.

4. **Architecture discovery is non-linear**: Expected HTTP, found PostgreSQL queue. Be ready to pivot.

### On Agent Architecture

5. **Hierarchical agents work**: Master orchestrator + specialized sub-agents scales complex projects.

6. **Completion reports enable handoff**: Standardized reports let next agent build on previous work.

7. **Anti-patterns are as important as patterns**: "DO NOT" sections prevent scope creep.

8. **Discovery before implementation**: "Find ChatContainer via CLAUDE.md" better than "ChatContainer is at X".

### On Implementation

9. **Adapter > Rewrite**: 27 features, 113 hours to rebuild. Adapter took 8 hours. 75% savings.

10. **Tool buffering enables workflows**: Intercept destructive calls, execute on approval. Preserves UX with new transport.

11. **Verbose prompts hurt tool calling**: AI SDK passes schemas automatically. Verbose prompts cause hallucination.

12. **Feature flags enable safety**: `NEXT_PUBLIC_USE_AI_SDK_CHAT` toggle allows instant rollback.

13. **Follow existing UI patterns**: `ValidationResults` mirrors `PlanChatMessage` pattern for consistency and faster implementation.

14. **Client-side state for UX features**: Live provider switching uses React state (not DB), preserving message history without server roundtrips.

---

*— End of Brainlift —*

*Last Updated: December 7, 2025*
*Research Documents: 22 files, 15,000+ lines*
*Total Implementation: ~6,955 lines across 8 PRs (PR1 → PR4)*
