# Chartsmith Architecture Decisions

**Project**: Chartsmith AI SDK Migration  
**Date**: December 2, 2025  
**Updated**: December 2, 2025 (added Decision 8, updated Decision 3 for full tool migration)  
**Scope**: PR1, PR1.5, and PR2 architectural choices

---

## Executive Summary

This document captures the architectural decisions made for migrating Chartsmith from its custom Anthropic SDK implementation to Vercel AI SDK. These decisions were driven by the project requirements:

- "Replace custom chat UI with Vercel AI SDK"
- "Migrate from direct @anthropic-ai/sdk usage to AI SDK Core"
- "Maintain all existing chat functionality (streaming, messages, history)"
- "All existing features continue to work (tool calling, file context, etc.)"

---

## Existing Chartsmith Architecture

Before documenting decisions, it's important to understand the current architecture:

```
┌─────────────────┐                              ┌─────────────────┐
│   React Chat    │      Server Actions          │  Next.js API    │
│   Components    │ ────────────────────────────►│    Routes       │
│                 │                              │                 │
│  Jotai Atoms    │                              │  Workspace APIs │
│  (Custom State) │                              └────────┬────────┘
└────────┬────────┘                                       │
         │                                                │ INSERT + pg_notify()
         │ WebSocket                                      │
         │ (Centrifugo)                                   ▼
         │                                       ┌─────────────────┐
         │◄──────────────────────────────────────│   PostgreSQL    │
         │        Real-time updates              │   work_queue    │
                                                 │   LISTEN/NOTIFY │
                                                 └────────┬────────┘
                                                          │ LISTEN
                                                 ┌────────▼────────┐
                                                 │   Go Backend    │
                                                 │   (Workers)     │
                                                 │                 │
                                                 │  Anthropic SDK  │
                                                 │  (All LLM calls)│
                                                 │                 │
                                                 │  3 Tools:       │
                                                 │  - text_editor  │
                                                 │  - latest_subchart_version │
                                                 │  - latest_kubernetes_version │
                                                 └─────────────────┘
```

**Key characteristics:**
- No `/api/chat` HTTP endpoint exists
- All LLM calls happen in Go via Anthropic SDK
- Communication via PostgreSQL LISTEN/NOTIFY, not HTTP
- Streaming via Centrifugo WebSocket, not SSE
- State managed via Jotai atoms
- 3 tools defined in Anthropic-native format in Go

---

## Decision 1: Hybrid Replacement with Deprecation

### Context

The requirement states "Replace custom chat UI with Vercel AI SDK." Two interpretations exist:

1. **Add alongside**: Create new AI SDK chat, keep old system functional
2. **True replacement**: New system becomes THE chat system

### Decision: New AI SDK System Becomes Primary; Legacy Deprecated

We chose a hybrid approach:
- Create new `/api/chat` endpoint using AI SDK
- New chat UI uses `useChat` hook
- Old chat path marked as deprecated
- Single user-facing chat experience

### Rationale

| Approach | Pros | Cons |
|----------|------|------|
| Add alongside | Low risk, quick | Doesn't meet "replace" requirement; two systems to maintain |
| True replacement | Meets requirement | Old system provides fallback during transition |

The hybrid approach satisfies "replace" (new system is primary) while maintaining a safety net (old code exists but unused).

### Implications

- Legacy `ChatContainer` receives deprecation banner
- All new development targets AI SDK path
- Old code remains for reference but is not user-facing
- Documentation clearly states new system is authoritative

---

## Decision 2: Go Handles No LLM Calls

### Context

Current Chartsmith has Go making all LLM calls via Anthropic SDK. The requirement to "migrate from @anthropic-ai/sdk" could mean:
- Only the npm package (TypeScript)
- All Anthropic usage including Go

### Decision: All LLM Calls Move to Node/AI SDK; Go Becomes Pure Application Logic

After PR1.5:
- **Node/AI SDK** = All "thinking" (LLM calls, tool orchestration, reasoning)
- **Go** = Pure application logic (chart CRUD, file operations, validation)

### Rationale

1. **Cleaner separation of concerns**: Intelligence layer vs. execution layer
2. **Simpler Go scope**: No LLM client modifications, just HTTP endpoints
3. **Spec compliance**: `@anthropic-ai/sdk` refers to the npm package; removing it from TypeScript satisfies the literal requirement
4. **Centralized provider config**: All model selection, API keys, and provider logic in one place (Node)

### What This Means for Go

| Before | After |
|--------|-------|
| Go calls Anthropic SDK directly | Go exposes HTTP endpoints |
| Go defines tools in Anthropic format | Tools defined in AI SDK format (Node) |
| Go orchestrates tool execution | Node orchestrates; Go executes |
| `pkg/llm/*` is active code | `pkg/llm/*` marked as legacy |

### Architecture Result

```
┌─────────────────────────────────────────────────────────────────┐
│                    NODE / AI SDK CORE                            │
│                    (All LLM "thinking")                          │
│                                                                  │
│  - streamText() for completions                                  │
│  - Tool definitions and orchestration                            │
│  - Provider selection (OpenRouter)                               │
│  - System prompts                                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ tool.execute() → HTTP
┌─────────────────────────────────────────────────────────────────┐
│                    GO BACKEND                                    │
│                    (Pure application logic)                      │
│                                                                  │
│  - Chart creation, loading, updating                             │
│  - File operations                                               │
│  - Validation execution (PR2)                                    │
│                                                                  │
│  NO LLM CALLS                                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Decision 3: Tool Migration Strategy

### Context

Chartsmith has 3 existing tools in Go (Anthropic-native format):

| Tool | Purpose | Location |
|------|---------|----------|
| `text_editor_20241022` | View, edit, create files | `pkg/llm/execute-action.go`, `pkg/agent/tools/text_editor.go` |
| `latest_subchart_version` | Look up subchart versions on ArtifactHub | `pkg/llm/conversational.go` |
| `latest_kubernetes_version` | Get current Kubernetes version info | `pkg/llm/conversational.go` |

### Decision: Port ALL Existing Tools for Full Feature Parity

**Ported to AI SDK (PR1.5):**
- `textEditor` - Essential for file operations in chat
- `createChart` - New tool for chart creation flow
- `getChartContext` - New tool for loading chart state
- `updateChart` - New tool for chart modifications
- `latestSubchartVersion` - **Ported for feature parity** (existing tool)
- `latestKubernetesVersion` - **Ported for feature parity** (existing tool)

### Rationale

1. **Requirement compliance**: The original requirement explicitly states "All existing features continue to work (tool calling, file context, etc.)" - this mandates porting ALL existing tools
2. **Demo requirement**: "Demonstrate creating a new chart via chat" requires `createChart`
3. **Complete feature parity**: Users should have identical capabilities in the new chat system
4. **No regressions**: Deferring utility tools would constitute a feature regression

### Tool Mapping

| Legacy Go Tool | AI SDK Tool | Status |
|----------------|-------------|--------|
| `text_editor_20241022` | `textEditor` | Ported in PR1.5 |
| `latest_subchart_version` | `latestSubchartVersion` | **Ported in PR1.5** |
| `latest_kubernetes_version` | `latestKubernetesVersion` | **Ported in PR1.5** |
| (new) | `createChart` | Created in PR1.5 |
| (new) | `getChartContext` | Created in PR1.5 |
| (new) | `updateChart` | Created in PR1.5 |

### Implications

- **6 tools total** in the new AI SDK chat system
- Full feature parity maintained - no capabilities lost
- Legacy Go tool code serves as implementation reference
- New AI SDK chat provides complete chart lifecycle AND utility capabilities

---

## Decision 4: RESTful Go HTTP Endpoints

### Context

AI SDK tools need to call Go for execution. What endpoint structure should we use?

### Decision: RPC-Style Endpoints with RESTful Verbs

**Endpoint Structure:**
```
POST  /api/tools/charts/create    → Create new chart
GET   /api/tools/charts/:id       → Load chart context
PATCH /api/tools/charts/:id       → Update chart
POST  /api/tools/editor           → File operations (view/create/edit)
POST  /api/validate               → Validation pipeline (PR2)
```

### Rationale

| Approach | Pros | Cons |
|----------|------|------|
| Single `/api/tools` endpoint | Simpler routing | Hard to debug; unclear what's happening |
| Pure REST (`/charts`, `/files`) | Standard | Doesn't group tool-related endpoints |
| RPC-style under `/api/tools/` | Clear purpose; grouped; debuggable | Slightly verbose |

The chosen approach:
- Groups all tool endpoints under `/api/tools/`
- Uses RESTful verbs where appropriate (GET for read, POST for create, PATCH for update)
- Makes logging and debugging clear ("POST to /api/tools/charts/create")
- Aligns with AI SDK tool naming conventions

### Server Configuration

- HTTP server runs on port 8080
- Starts alongside existing PostgreSQL queue listener
- Environment variable: `GO_BACKEND_URL=http://localhost:8080`

---

## Decision 5: Non-Streaming Communication with Go

### Context

Current Go→Anthropic communication uses streaming. Should Node→Go tool calls use streaming?

### Decision: JSON Request/Response (No Streaming)

All communication between Node and Go uses synchronous JSON:
- Tool calls Go endpoint with JSON body
- Go executes operation
- Go returns complete JSON response
- No streaming, no chunked responses

### Rationale

1. **Go doesn't need streaming**: Go pushes results to Centrifugo for real-time updates; it doesn't stream to the browser directly
2. **Simpler implementation**: HTTP request/response is easier than managing streams
3. **Fewer failure modes**: No partial responses, connection management, or stream parsing
4. **Frontend streaming unaffected**: `useChat` handles browser streaming via AI SDK's data stream protocol

### Where Streaming Exists

| Path | Streaming? | Protocol |
|------|------------|----------|
| Browser ↔ `/api/chat` | Yes | AI SDK Data Stream |
| `/api/chat` ↔ OpenRouter | Yes | SSE |
| `/api/chat` ↔ Go endpoints | No | JSON request/response |
| Go ↔ Centrifugo | N/A | Push events |

---

## Decision 6: 3-PR Structure

### Context

Original plan had 2 PRs. Analysis revealed PR1 would be overloaded with both foundation AND feature parity work.

### Decision: Split into PR1 (Foundation) + PR1.5 (Parity) + PR2 (Validation)

**PR1: AI SDK Foundation**
- New `/api/chat` endpoint with `streamText`
- `useChat` hook integration
- Provider selection UI
- Basic streaming chat working
- No tools, no Go changes

**PR1.5: Migration & Feature Parity**
- Shared LLM client library
- Go HTTP server and tool endpoints
- AI SDK tools (createChart, getChartContext, updateChart, textEditor)
- System prompt migration
- Remove @anthropic-ai/sdk from TypeScript
- Deprecation of legacy chat

**PR2: Validation Agent**
- Go validation pipeline (helm lint, template, kube-score)
- `validateChart` tool
- Live provider switching
- Validation results UI

### Rationale

1. **Clear deliverables**: Each PR has testable, demonstrable outcomes
2. **Risk isolation**: PR1 validates AI SDK works before adding complexity
3. **PR1.5 focuses on migration**: Pure feature parity work, not mixed with new features
4. **PR2 builds on infrastructure**: Validation uses proven tool→Go HTTP pattern

### Dependencies

```
PR1 ──► PR1.5 ──► PR2
        │
        └── PR1.5 creates the tool infrastructure PR2 needs
```

---

## Decision 7: System Prompt Location

### Context

Current system prompts live in `pkg/llm/system.go` (Go). With LLM calls moving to Node, where should prompts live?

### Decision: Migrate Prompts to Node (`lib/ai/prompts.ts`)

System prompts move to TypeScript:
- Create `getSystemPrompt()` function
- Accept context (workspaceId, revisionNumber, chartContext)
- Document available tools and usage guidance
- Include new chart mode behavior (revisionNumber === 0)

### Rationale

1. **Colocation**: Prompts should be near the LLM call site
2. **Tool awareness**: Prompts reference AI SDK tools, which are defined in Node
3. **Dynamic context**: Easier to inject workspace/chart context in Node
4. **Single source**: Avoids prompt duplication across languages

### Migration Approach

- Copy relevant content from `pkg/llm/system.go`
- Adapt for AI SDK tool names and behaviors
- Add tool invocation guidance
- Keep Go prompts as reference (legacy)

---

## Decision 8: HTTP vs PostgreSQL Queue for Tool Communication

### Context

The existing Chartsmith architecture uses PostgreSQL LISTEN/NOTIFY for communication between the frontend and Go backend. When implementing AI SDK tools, we needed to decide how tools should communicate with Go:

1. **Option A**: Add HTTP endpoints to Go (new pattern)
2. **Option B**: Use existing PostgreSQL queue pattern (async)
3. **Option C**: Hybrid - Next.js spawns Go CLI

### Decision: HTTP Endpoints (Option A)

AI SDK tools communicate with Go via synchronous HTTP request/response:
- Tool `execute()` calls Go endpoint with JSON body
- Go processes request and returns JSON response
- No streaming, no queue, no async complexity

### Rationale

| Aspect | PostgreSQL Queue | HTTP |
|--------|------------------|------|
| Latency | Higher (queue + notify + consume) | Lower (direct request/response) |
| Complexity | Requires queue handlers, Centrifugo integration | Simple `fetch()` calls |
| Use case | Long-running async work | Synchronous tool execution |
| Error handling | Harder to propagate errors to caller | Natural HTTP status codes |
| Existing pattern | Used for workspace operations | New, but industry-standard |

**Why HTTP is better for AI SDK tools:**

1. **Synchronous responses required**: Tools need to return results for the LLM to continue reasoning
2. **Stream continuity**: Tool results must flow back in the same AI SDK data stream
3. **Simpler debugging**: HTTP requests are easy to trace and test
4. **Industry standard**: HTTP is the standard pattern for tool execution in AI agents
5. **No async complexity**: PostgreSQL queue introduces unnecessary complexity for synchronous operations

### Implications

- New HTTP server runs alongside existing PostgreSQL listener on port 8080
- Environment variable: `GO_BACKEND_URL=http://localhost:8080`
- All tool endpoints use JSON request/response
- Existing PostgreSQL/Centrifugo patterns remain for workspace events

---

## Final Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    REACT CHAT UI                                 │
│                    (useChat hook)                                │
│                                                                  │
│  - Single chat interface (legacy deprecated)                     │
│  - Provider selector                                             │
│  - Streaming message display                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ POST /api/chat (streaming)
┌─────────────────────────────────────────────────────────────────┐
│                    NODE / AI SDK CORE                            │
│                                                                  │
│  lib/ai/llmClient.ts     - Shared LLM client, provider factory   │
│  lib/ai/prompts.ts       - System prompts                        │
│  lib/ai/tools/*.ts       - Tool definitions (6 total)            │
│  lib/ai/tools/utils.ts   - Shared error handling                 │
│                                                                  │
│  Tools (PR1.5):                                                  │
│  - createChart             → POST /api/tools/charts/create       │
│  - getChartContext         → GET  /api/tools/charts/:id          │
│  - updateChart             → PATCH /api/tools/charts/:id         │
│  - textEditor              → POST /api/tools/editor              │
│  - latestSubchartVersion   → POST /api/tools/versions/subchart   │
│  - latestKubernetesVersion → POST /api/tools/versions/kubernetes │
│                                                                  │
│  Tools (PR2):                                                    │
│  - validateChart           → POST /api/validate                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ HTTP (JSON, non-streaming)
┌─────────────────────────────────────────────────────────────────┐
│                    GO BACKEND                                    │
│                    (Pure application logic)                      │
│                                                                  │
│  pkg/api/server.go            - HTTP server on :8080             │
│  pkg/api/errors.go            - Standardized error responses     │
│  pkg/api/handlers/charts.go   - Chart CRUD operations            │
│  pkg/api/handlers/editor.go   - File operations                  │
│  pkg/api/handlers/versions.go - Version lookups (ArtifactHub, K8s)│
│  pkg/validation/*             - Validation pipeline (PR2)        │
│                                                                  │
│  NO LLM CALLS                                                    │
│                                                                  │
│  Legacy (deprecated, unused):                                    │
│  - pkg/llm/client.go                                             │
│  - pkg/llm/conversational.go                                     │
│  - pkg/llm/execute-action.go                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DATA LAYER                                    │
│                                                                  │
│  - PostgreSQL (workspaces, charts, revisions)                    │
│  - File system (chart files)                                     │
│  - Centrifugo (real-time updates via pg_notify)                  │
│  - External APIs (ArtifactHub, Kubernetes releases)              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary: Decisions at a Glance

| # | Decision | Choice |
|---|----------|--------|
| 1 | Parallel vs. Replacement | Hybrid: New system primary, legacy deprecated |
| 2 | Go LLM Responsibility | None: Go becomes pure application logic |
| 3 | Tool Migration | **Port ALL existing tools for full feature parity (6 tools total)** |
| 4 | Go Endpoint Design | RPC-style under `/api/tools/` with RESTful verbs |
| 5 | Go Communication | Non-streaming JSON request/response |
| 6 | PR Structure | 3 PRs: Foundation → Parity → Validation |
| 7 | System Prompt Location | Migrate to Node (`lib/ai/prompts.ts`) |
| 8 | Tool↔Go Communication | **HTTP endpoints (not PostgreSQL queue) for synchronous tool execution** |

---

*End of Document*
