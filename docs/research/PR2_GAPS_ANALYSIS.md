# PR2 PRD Gap Analysis

**Purpose**: Document identified gaps in PR2 PRDs that require verification via codebase exploration before finalizing specifications.

**Status**: ✅ EXPLORATION COMPLETE - 2024-12-01

**Research Summary**: Chartsmith uses a **database-driven architecture** (NOT HTTP REST between Next.js and Go). Both services communicate via PostgreSQL LISTEN/NOTIFY + Centrifugo WebSockets. This fundamentally changes the PR2 approach.

---

## PR1 Alignment Status: ✅ Confirmed Aligned

| Aspect | Product PRD | Tech PRD | Architecture Decisions | Status |
|--------|-------------|----------|------------------------|--------|
| Provider approach | OpenRouter | OpenRouter | ADR-002 OpenRouter | ✅ Consistent |
| Provider selector | Per-conversation, hidden after first msg | Same | ADR-003 phased approach | ✅ Consistent |
| Dependencies | ai, @ai-sdk/react, @openrouter | Same + versions | Same | ✅ Consistent |
| Go backend changes | None in PR1 | Deferred to PR2 | ADR-004, ADR-006 | ✅ Consistent |
| API contract | POST /api/chat | Same with details | Same | ✅ Consistent |
| Default provider | GPT-4o | GPT-4o | ADR-007 GPT-4o | ✅ Consistent |

**No action required for PR1.**

---

## PR2 Alignment Status: ⚠️ Mostly Aligned, Gaps Identified

| Aspect | Product PRD | Tech PRD | Supplemental | Status |
|--------|-------------|----------|--------------|--------|
| Pipeline stages | lint→template→kube-score | Same | Same | ✅ Consistent |
| kube-score failure | Non-fatal | Non-fatal | N/A | ✅ Consistent |
| Go package structure | N/A | pkg/validation/ | Same | ✅ Consistent |
| Tool name | validateChart | validateChart | validateChart | ✅ Consistent |
| API endpoint | POST /api/validate | Same | Same | ✅ Consistent |
| LiveProviderSwitcher | Always visible in header | Same | Same | ✅ Consistent |

---

## 🔴 Critical Gaps (Must Address)

### Gap 1: Next.js ↔ Go Backend Communication

**Status**: ⚠️ ARCHITECTURE MISMATCH - PRDs need revision

**The Problem**:
The architecture shows Next.js API calling Go backend at `/api/validate`, but nowhere do we specify:
- Is Go a separate service on a different port?
- What's the URL? (`GO_BACKEND_URL` referenced but never defined)
- How does this work in local dev vs production?
- Is there authentication between Next.js and Go?
- Do they share a process or are they separate services?

**FINDING: Database-Driven Architecture (NOT HTTP)**

Chartsmith does **NOT** use HTTP REST between Next.js and Go. Instead:

```
┌─────────────────┐                    ┌─────────────────┐
│   Next.js App   │                    │   Go Worker     │
│   (port 3000)   │                    │   (no HTTP)     │
└────────┬────────┘                    └────────┬────────┘
         │                                      │
         │ INSERT + pg_notify()                 │ LISTEN
         │                                      │
         └──────────────┬───────────────────────┘
                        │
              ┌─────────▼─────────┐
              │   PostgreSQL      │
              │   (port 5432)     │
              │   + work_queue    │
              └─────────┬─────────┘
                        │
              ┌─────────▼─────────┐
              │   Centrifugo      │
              │   (port 8000)     │
              │   WebSocket PubSub│
              └───────────────────┘
```

**Key Files**:
- `chartsmith-app/lib/utils/queue.ts:9-20` - Enqueue work via INSERT + pg_notify()
- `chartsmith/pkg/listener/start.go:13-119` - Go LISTEN handlers
- `chartsmith/pkg/realtime/centrifugo.go:48-66` - Real-time updates

**Environment Variables**:
- `CHARTSMITH_PG_URI` - PostgreSQL connection (shared by both services)
- `CHARTSMITH_CENTRIFUGO_ADDRESS` - Real-time messaging server
- `CHARTSMITH_CENTRIFUGO_API_KEY` - Centrifugo API auth
- **NO `GO_BACKEND_URL`** - Services don't communicate via HTTP

**Resolution Required**:
- ❌ PRDs assume HTTP REST - needs architectural revision
- Option A: Add HTTP endpoint to Go backend (new pattern)
- Option B: Use existing queue pattern (validation as async job)
- **Recommended**: Option A for synchronous validation response

---

### Gap 2: Chart Path Resolution

**Status**: ✅ RESOLVED - Workspace-centric model with "first chart wins"

**The Problem**:
When user says "validate my chart", the `validateChart` tool requires `chartPath` as input. But:
- How does the AI know which chart to validate?
- Is there a "current chart in context" concept in Chartsmith?
- Does the user explicitly provide the path, or is it inferred?
- What if multiple charts exist?

**FINDING: Workspace-Centric Model**

Chartsmith uses a **workspace-centric model** where charts exist within a workspace at a specific revision. There is **no explicit "current chart" tracking**. Instead:

1. **Workspace contains charts**: `Workspace.Charts[]` array
2. **First chart wins**: Operations default to `workspace.Charts[0]`
3. **Files know their chart**: Each file has `chart_id` in database
4. **Chat is workspace-scoped**: Messages linked to `workspace_id`, not `chart_id`

**Key Implementation** (`pkg/listener/execute-plan.go:78-105`):
```go
var chartID *string
if len(w.Charts) > 0 {
    chartID = &w.Charts[0].ID  // Always uses first chart
}
```

**Key Files**:
- `pkg/workspace/types/types.go:32-46` - Workspace struct with Charts array
- `chartsmith-app/atoms/workspace.ts:6` - Frontend workspace atom (no selectedChart)
- `pkg/listener/execute-plan.go:78-80` - First chart selection pattern

**Resolution for validateChart Tool**:
- ✅ Use `workspace.Charts[0].ID` to determine chart context
- ✅ Chart files are accessible via workspace + revision
- ✅ No explicit chartPath input needed - derive from workspace context
- **Recommended**: Tool gets workspace ID from conversation context, uses first chart

---

### Gap 3: Existing Tools Preservation

**Status**: ✅ RESOLVED - 3 tools exist, all in Go backend using Anthropic-native format

**The Problem**:
PR1 Product PRD states:
> "File context tools work as before"
> "Chart generation tools function correctly"

But we never identified what tools currently exist. When PR2 adds `validateChart` to the `tools` object in `streamText`, we must include existing tools too.

**FINDING: Anthropic-Native Tools in Go Backend**

Chartsmith uses **Anthropic-native tool_use** (NOT Vercel AI SDK). 3 tools exist, all defined inline in Go:

| Tool | Location | Purpose |
|------|----------|---------|
| `text_editor_20241022` | `pkg/llm/execute-action.go:510-531` | File ops (view, str_replace, create) |
| `latest_subchart_version` | `pkg/llm/conversational.go:100-113` | Query ArtifactHub for chart versions |
| `latest_kubernetes_version` | `pkg/llm/conversational.go:114-127` | Return K8s version strings |

**Tool Definition Format** (Go inline JSON schema):
```go
tools := []anthropic.ToolParam{
    {
        Name: anthropic.F("text_editor_20241022"),
        InputSchema: anthropic.F(interface{}(map[string]interface{}{
            "type": "object",
            "properties": map[string]interface{}{
                "command": map[string]interface{}{
                    "type": "string",
                    "enum": []string{"view", "str_replace", "create"},
                },
                "path": map[string]interface{}{"type": "string"},
                // ...
            },
        })),
    },
}
```

**Key Files**:
- `pkg/llm/execute-action.go:510-531` - text_editor tool definition
- `pkg/llm/conversational.go:99-128` - conversational tools
- `pkg/llm/client.go:12-21` - Anthropic SDK client

**Resolution for PR2**:
- ⚠️ PRDs assume Vercel AI SDK - codebase uses direct Anthropic SDK
- ✅ Existing tools are backend-only (no frontend tool definitions)
- ✅ Add `validateChart` as 4th tool in Go backend
- **Recommended**: Follow existing pattern - add tool to appropriate handler file

---

## 🟡 Medium Gaps (Should Address)

### Gap 4: System Prompt for Tool Usage

**Status**: ✅ RESOLVED - System prompts in Go backend at pkg/llm/system.go

**The Problem**:
The AI needs guidance on when to invoke `validateChart`. Current PRDs say:
> "AI recognizes validation intent from context"

But there's no system prompt update specified. The tool `description` field alone may not reliably trigger invocation.

**FINDING: Comprehensive System Prompts in Go Backend**

System prompts are defined in `pkg/llm/system.go` as string constants:

| Prompt | Line | Purpose |
|--------|------|---------|
| `commonSystemPrompt` | 20-55 | Base ChartSmith identity and constraints |
| `chatOnlySystemPrompt` | 57-65 | Extends common for conversational Q&A |
| `executePlanSystemPrompt` | 111-120 | For file execution with text_editor |
| `detailedPlanSystemPrompt` | 89-98 | For creating action plans |
| `initialPlanSystemPrompt` | 67-76 | For new chart creation |

**Context Injection Mechanisms**:
1. **Chart structure** injected via `getChartStructure()` at `conversational.go:236`
2. **Relevant files** selected via vector embeddings at `context.go:28-175`
3. **Chat history** included from `workspace.ListChatMessagesAfterPlan()`
4. **Plan descriptions** added to messages before execution

**Key Files**:
- `pkg/llm/system.go` - All system prompt constants
- `pkg/llm/create-knowledge.go:3-52` - Instruction blocks
- `pkg/workspace/context.go:28-175` - Vector-based file selection

**Resolution for validateChart**:
- ✅ Add validation instructions to `chatOnlySystemPrompt` or `commonSystemPrompt`
- ✅ Tool description + system prompt guidance will trigger invocation
- **Recommended**: Add to `chatOnlySystemPrompt` since validation is conversational

---

### Gap 5: AI Interpretation Flow After Tool Execution

**Status**: ✅ RESOLVED - Manual tool loop pattern in Go backend

**The Problem**:
After `validateChart` returns results, the PRDs describe the UX (AI explains issues) but not the mechanism:
- Does the AI automatically interpret results?
- Is this built into Vercel AI SDK behavior?
- Do we need to configure anything?

**FINDING: Manual Agentic Loop (NOT Automatic)**

Chartsmith uses **manual tool result construction** with the Anthropic SDK. The pattern:

```go
// 1. Call Claude with tools
stream := client.Messages.NewStreaming(ctx, params)

// 2. Check for tool_use in response
for _, block := range message.Content {
    if block.Type == ContentBlockTypeToolUse {
        // 3. Execute tool locally
        result := executeToolLocally(block.Input)

        // 4. Construct tool_result manually
        toolResults = append(toolResults,
            anthropic.NewToolResultBlock(block.ID, result, false))
    }
}

// 5. Send results back as user message
messages = append(messages, anthropic.MessageParam{
    Role:    anthropic.F(anthropic.MessageParamRoleUser),
    Content: anthropic.F(toolResults),
})

// 6. Loop continues until no more tool calls
```

**Key Files**:
- `pkg/llm/execute-action.go:542-673` - Main tool execution loop
- `pkg/llm/conversational.go:135-230` - Conversational tool loop
- `pkg/realtime/centrifugo.go:48-66` - Real-time updates to frontend

**Real-time Updates via Centrifugo**:
- Tool results DO NOT stream directly to frontend
- Centrifugo WebSocket pushes `chatmessage-updated` events
- Frontend receives via `useCentrifugo.ts:452-453`

**Resolution for validateChart**:
- ✅ Follow existing manual loop pattern
- ✅ Execute validation, construct tool_result, append to messages
- ✅ Claude will interpret results on next iteration
- ✅ Stream interpretation via Centrifugo to frontend

---

### Gap 6: Error Message Mapping

**The Problem**:
Product PRD lists user-friendly error messages. Tech PRD lists technical error scenarios. They don't explicitly map to each other.

**Tech PRD Error Scenarios**:
| Scenario | Technical Response |
|----------|-------------------|
| helm not installed | 500 with "helm CLI not found" |
| kube-score not installed | Continue without, log warning |
| Chart path doesn't exist | 400 with "chart path does not exist" |
| Validation timeout | 504 with "validation timed out" |

**Product PRD Error UX**:
| Status | Display |
|--------|---------|
| Error | "Error: [message]" with retry button |

**Missing**: Mapping from technical errors to user-friendly messages.

**Resolution**: Add error message mapping table to Tech PRD:
```
| Technical Error | User-Facing Message |
|-----------------|---------------------|
| "helm CLI not found" | "Validation tools are not available. Please contact support." |
| "chart path does not exist" | "Could not find the chart to validate. Please check the path." |
| "validation timed out" | "Validation is taking too long. Please try again." |
```

---

## 🟢 Minor Gaps (Nice to Address)

### Gap 7: Test Chart Fixtures

**Status**: ✅ RESOLVED - 3 test chart locations exist with multiple fixtures

**The Problem**:
PRDs reference `test_chart/` but don't specify what test scenarios we need.

**FINDING: Comprehensive Test Fixtures Exist**

| Location | Contents |
|----------|----------|
| `test_chart/test-chart/` | Complete basic Helm chart (12 files) |
| `testdata/charts/` | Packaged chart archives (4 .tgz files) |
| `bootstrap/default-workspace/charts/new-chart/` | Bootstrap template (7 files) |

**Test Chart Details** (`test_chart/test-chart/Chart.yaml`):
```yaml
apiVersion: v2
name: test-chart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

**Packaged Charts Available**:
- `testdata/charts/empty-chart-0.1.0.tgz` - Empty chart
- `testdata/charts/okteto-1.29.0-rc.2.tgz` - Integration test chart
- `testdata/charts/sourcegraph-6.1.5633.tgz` - Real-world complex chart

**K8s Manifests for Conversion Testing**:
- `testdata/k8s/simple-app/` - deployment.yaml, service.yaml, configmap.yaml

**Key Files**:
- `pkg/integration/apply-changes_chart.go` - Uses okteto chart
- `pkg/testhelpers/pg.go` - Test database setup
- `cmd/test-data.go` - Test data generation

**Resolution for PR2 Validation Testing**:
- ✅ Use `test_chart/test-chart/` for basic validation tests
- ✅ Can create additional fixtures for error scenarios
- **Recommended**: Add lint-error and kube-score-warning fixtures

---

### Gap 8: Provider Switch During Active Validation

**The Problem**:
What happens if user switches provider while validation tool is executing?

**Expected Behavior** (needs confirmation):
1. Current tool call completes with original provider
2. Tool result interpretation uses original provider
3. Provider switch takes effect on next user message

**Edge Case**:
User switches provider → validation completes → AI interprets with... which provider?

**Resolution**: Add clarifying note to Tech PRD:
> "Provider switches take effect on the next user-initiated message. In-flight tool calls and their interpretations complete with the provider that was active when the tool was invoked."

---

## Summary: Gaps by Priority

| Gap | Severity | Status | Key Finding |
|-----|----------|--------|-------------|
| Next.js ↔ Go communication | 🔴 Critical | ⚠️ ARCHITECTURE MISMATCH | Database-driven, NOT HTTP REST |
| Chart path resolution | 🔴 Critical | ✅ RESOLVED | Workspace-centric, first chart wins |
| Existing tools preservation | 🔴 Critical | ✅ RESOLVED | 3 tools in Go backend, Anthropic-native |
| System prompt for tools | 🟡 Medium | ✅ RESOLVED | pkg/llm/system.go has all prompts |
| AI interpretation flow | 🟡 Medium | ✅ RESOLVED | Manual tool loop pattern |
| Error message mapping | 🟡 Medium | ⏳ PENDING | Still needs mapping table |
| Test fixtures | 🟢 Minor | ✅ RESOLVED | 3 locations with comprehensive fixtures |
| Provider switch during validation | 🟢 Minor | ⏳ PENDING | Still needs clarifying note |

---

## Exploration Checklist - COMPLETED ✅

### Critical Questions
- [x] How does Next.js app communicate with Go backend? → **PostgreSQL LISTEN/NOTIFY + Centrifugo WebSocket**
- [x] Is there a "current chart" context mechanism? → **Workspace.Charts[0] (first chart wins)**
- [x] What tools exist today? → **3 tools: text_editor, latest_subchart_version, latest_kubernetes_version**

### Medium Questions
- [x] Where are system prompts defined? → **pkg/llm/system.go**
- [x] How do existing tool results get interpreted? → **Manual agentic loop with tool_result blocks**

### Minor Questions
- [x] What test charts exist? → **test_chart/, testdata/charts/, bootstrap/**
- [ ] What error handling patterns exist? → **Partial - see individual gap findings**

---

## 🚨 CRITICAL PRD REVISION REQUIRED

The exploration revealed that **PRDs assume HTTP REST architecture** but Chartsmith uses **database-driven async communication**. Two options:

### Option A: Add HTTP Endpoint (Recommended for Synchronous Response)
- Add new HTTP handler to Go backend
- Next.js calls Go directly for validation
- Returns results synchronously
- **Pro**: Clean integration with AI SDK tools
- **Con**: New pattern in codebase

### Option B: Use Existing Queue Pattern (Async)
- Enqueue `validate_chart` job via pg_notify
- Go worker processes, publishes results via Centrifugo
- Frontend receives via WebSocket
- **Pro**: Consistent with existing architecture
- **Con**: More complex UX for synchronous-feeling validation

**Recommended**: Option A - Add minimal HTTP endpoint for validation

---

## Next Steps

1. ❌ **Revise Tech PRD** - Update architecture to reflect database-driven pattern OR add HTTP endpoint decision
2. ✅ **Use existing tool format** - Add validateChart in Go backend following existing pattern
3. ✅ **Update system prompts** - Add validation instructions to chatOnlySystemPrompt
4. ✅ **Leverage test fixtures** - Use test_chart/ for validation testing
5. ⏳ **Add error mapping table** - Map technical errors to user messages

---

*Document Updated - PR2 Gap Analysis - Exploration Complete 2024-12-01*
