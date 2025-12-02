# PR2 Codebase Research Findings

---
date: 2024-12-01T21:00:00-08:00
researcher: Claude Code
git_commit: e615830f7b0b60ecb817d762b9b4e4186f855d85
branch: main
repository: UnchartedTerritory/chartsmith
topic: "PR2 Validation Agent - Codebase Architecture Analysis"
tags: [research, codebase, chartsmith, validation, helm, architecture]
status: complete
last_updated: 2024-12-01
last_updated_by: Claude Code
---

## Research Question

Answer the gap analysis questions for PR2 (validateChart tool implementation):

1. How does Next.js communicate with Go backend? (ports, proxy, env vars)
2. Is there a "current chart" context mechanism?
3. What tools exist today and where are they defined?
4. Where are system prompts defined?
5. How do tool results flow back to the LLM?
6. What test charts exist in test_chart/ or testdata/?

## Executive Summary

Chartsmith uses a **database-driven architecture** where Next.js and Go communicate via PostgreSQL LISTEN/NOTIFY, NOT HTTP REST. This fundamentally differs from the PR2 PRD assumptions. The codebase has:

- **3 existing AI tools** defined in Go backend using Anthropic-native format
- **Workspace-centric model** where `Charts[0]` is implicitly the "current chart"
- **System prompts** in `pkg/llm/system.go` with vector-based context injection
- **Manual tool loop** pattern for tool result handling
- **Comprehensive test fixtures** across 3 directories

---

## 1. Next.js ↔ Go Backend Communication

### Architecture: Database-Driven (NOT HTTP REST)

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

### Key Files

| File | Purpose |
|------|---------|
| `chartsmith-app/lib/utils/queue.ts:9-20` | Enqueue work via INSERT + pg_notify() |
| `chartsmith/pkg/listener/start.go:13-119` | Go LISTEN handlers for channels |
| `chartsmith/pkg/persistence/pg.go:24-57` | Go PostgreSQL connection pool |
| `chartsmith-app/lib/data/db.ts:1-38` | Next.js PostgreSQL pool |
| `chartsmith/pkg/realtime/centrifugo.go:48-66` | Real-time event publishing |

### Environment Variables

| Variable | Service | Purpose |
|----------|---------|---------|
| `CHARTSMITH_PG_URI` | Both | PostgreSQL connection string |
| `CHARTSMITH_CENTRIFUGO_ADDRESS` | Both | Real-time messaging server URL |
| `CHARTSMITH_CENTRIFUGO_API_KEY` | Go | Centrifugo API authentication |
| `ANTHROPIC_API_KEY` | Go | LLM API key |
| `NEXT_PUBLIC_CENTRIFUGO_ADDRESS` | Next.js | WebSocket endpoint for frontend |

### Work Queue Pattern

```typescript
// chartsmith-app/lib/utils/queue.ts:9-20
export async function enqueueWork(channel: string, payload: QueuePayload): Promise<void> {
  const client = getDB(await getParam("DB_URI"));
  const id = srs.default({ length: 12, alphanumeric: true });

  // 1. Insert work into queue table
  await client.query(
    `INSERT INTO work_queue (id, channel, payload, created_at) VALUES ($1, $2, $3, NOW())`,
    [id, channel, payload]
  );

  // 2. Notify Go worker via PostgreSQL LISTEN/NOTIFY
  await client.query(`SELECT pg_notify('${channel}', $1)`, [id]);
}
```

### Existing Channels

| Channel | Handler Location | Purpose |
|---------|------------------|---------|
| `new_intent` | `pkg/listener/new_intent.go` | Chat intent classification |
| `new_plan` | `pkg/listener/new-plan.go` | Plan creation |
| `execute_plan` | `pkg/listener/execute-plan.go` | Plan execution |
| `render_workspace` | `pkg/listener/render-workspace.go` | Helm template rendering |
| `new_conversion` | `pkg/listener/new-conversion.go` | K8s to Helm conversion |
| `publish_workspace` | `pkg/listener/publish-workspace.go` | Chart publishing |

---

## 2. Chart Path/Context Mechanism

### Workspace-Centric Model

Chartsmith uses workspaces, not direct chart paths. Each workspace contains:

```go
// pkg/workspace/types/types.go:32-46
type Workspace struct {
    ID                       string    `json:"id"`
    CurrentRevision          int       `json:"current_revision"`
    IncompleteRevisionNumber *int      `json:"incomplete_revision_number,omitempty"`
    Charts                   []Chart   `json:"charts"`
    Files                    []File    `json:"files"`
    CurrentPlans             []Plan    `json:"current_plans"`
    PreviousPlans            []Plan    `json:"previous_plans"`
}
```

### "First Chart Wins" Pattern

```go
// pkg/listener/execute-plan.go:78-105
var chartID *string
if len(w.Charts) > 0 {
    chartID = &w.Charts[0].ID  // Always uses first chart
}

llm.CreateExecutePlan(ctx, ..., w, plan, &w.Charts[0], relevantFiles)
```

### File Association

Files track their chart via database column:

```go
// pkg/workspace/types/types.go:7-15
type File struct {
    ID             string  `json:"id"`
    RevisionNumber int     `json:"revision_number"`
    ChartID        string  `json:"chart_id,omitempty"`
    WorkspaceID    string  `json:"workspace_id"`
    FilePath       string  `json:"filePath"`
    Content        string  `json:"content"`
}
```

### Frontend State (Jotai)

```typescript
// chartsmith-app/atoms/workspace.ts:6
export const workspaceAtom = atom<Workspace | null>(null)

// Line 101-105: Derived charts atom
export const chartsAtom = atom(get => {
  const workspace = get(workspaceAtom)
  if (!workspace) return []
  return workspace.charts
})
```

**Note**: No `selectedChartAtom` exists - the frontend doesn't track active chart selection.

---

## 3. Existing AI Tools

### Tool Inventory

| Tool Name | Location | Purpose |
|-----------|----------|---------|
| `text_editor_20241022` | `pkg/llm/execute-action.go:510-531` | File operations (view, str_replace, create) |
| `latest_subchart_version` | `pkg/llm/conversational.go:100-113` | Query ArtifactHub for chart versions |
| `latest_kubernetes_version` | `pkg/llm/conversational.go:114-127` | Return K8s version strings |

### Tool Definition Format (Anthropic-Native)

```go
// pkg/llm/execute-action.go:510-531
tools := []anthropic.ToolParam{
    {
        Name: anthropic.F(TextEditor_Sonnet35),  // "text_editor_20241022"
        InputSchema: anthropic.F(interface{}(map[string]interface{}{
            "type": "object",
            "properties": map[string]interface{}{
                "command": map[string]interface{}{
                    "type": "string",
                    "enum": []string{"view", "str_replace", "create"},
                },
                "path": map[string]interface{}{
                    "type": "string",
                },
                "old_str": map[string]interface{}{
                    "type": "string",
                },
                "new_str": map[string]interface{}{
                    "type": "string",
                },
            },
        })),
    },
}
```

### Text Editor Commands

| Command | Purpose | Implementation |
|---------|---------|----------------|
| `view` | View file contents | Returns content or error if missing |
| `str_replace` | Replace string in file | Exact match, then fuzzy with 10s timeout |
| `create` | Create new file | Fails if file exists |

### SDK Used

- **Go Backend**: `github.com/anthropics/anthropic-sdk-go`
- **NOT Vercel AI SDK**: No `ai` or `@ai-sdk` packages in frontend

---

## 4. System Prompts

### Location: `pkg/llm/system.go`

| Prompt Constant | Lines | Purpose |
|-----------------|-------|---------|
| `endUserSystemPrompt` | 3-18 | For chart operators (values.yaml only) |
| `commonSystemPrompt` | 20-55 | Base ChartSmith identity |
| `chatOnlySystemPrompt` | 57-65 | Conversational Q&A |
| `initialPlanSystemPrompt` | 67-76 | New chart creation |
| `updatePlanSystemPrompt` | 78-87 | Plan updates |
| `detailedPlanSystemPrompt` | 89-98 | Action plan creation |
| `executePlanSystemPrompt` | 111-120 | File execution |
| `convertFileSystemPrompt` | 122-140 | K8s manifest conversion |

### Context Injection

1. **Chart Structure**: `getChartStructure()` at `conversational.go:236`
2. **Relevant Files**: Vector embeddings via `ChooseRelevantFilesForChatMessage()` at `context.go:28-175`
3. **Chat History**: `workspace.ListChatMessagesAfterPlan()`
4. **Plan Descriptions**: Added before execution

### Vector-Based File Selection

```go
// pkg/workspace/context.go:89-117
query := `
    WITH similarities AS (
        SELECT id, file_path, content, embeddings,
               1 - (embeddings <=> $1) as similarity
        FROM workspace_file
        WHERE workspace_id = $2 AND revision_number = $3
        AND embeddings IS NOT NULL
    )
    SELECT * FROM similarities ORDER BY similarity DESC
`
```

---

## 5. Tool Results Flow

### Manual Agentic Loop Pattern

```go
// pkg/llm/execute-action.go:542-673 (simplified)
for {
    // 1. Stream response from Claude
    stream := client.Messages.NewStreaming(ctx, params)

    // 2. Accumulate message
    for stream.Next() {
        message.Accumulate(event)
    }

    // 3. Check for tool calls
    for _, block := range message.Content {
        if block.Type == ContentBlockTypeToolUse {
            // 4. Execute tool locally
            result := executeToolLocally(block.Input)

            // 5. Construct tool_result
            toolResults = append(toolResults,
                anthropic.NewToolResultBlock(block.ID, result, false))
        }
    }

    // 6. No tool calls = done
    if len(toolResults) == 0 {
        break
    }

    // 7. Send results back as user message
    messages = append(messages, anthropic.MessageParam{
        Role:    anthropic.F(anthropic.MessageParamRoleUser),
        Content: anthropic.F(toolResults),
    })
}
```

### Real-time Updates to Frontend

```go
// pkg/realtime/centrifugo.go:48-66
func SendEvent(ctx context.Context, r types.Recipient, e types.Event) error {
    // Publish to Centrifugo channel
    userChannelName := fmt.Sprintf("%s#%s", e.GetChannelName(), userID)
    return sendMessage(userChannelName, messageData)
}
```

### Frontend WebSocket Handler

```typescript
// chartsmith-app/hooks/useCentrifugo.ts:443-466
sub.on("publication", (ctx) => {
    switch (ctx.data.eventType) {
        case "chatmessage-updated":
            handleChatMessageUpdated(ctx.data);
            break;
        case "artifact-updated":
            handleArtifactUpdated(ctx.data);
            break;
        case "plan-updated":
            handlePlanUpdated(ctx.data);
            break;
    }
});
```

---

## 6. Test Chart Fixtures

### Directory Locations

| Location | Type | Contents |
|----------|------|----------|
| `test_chart/test-chart/` | Unpacked | Complete basic Helm chart (12 files) |
| `testdata/charts/` | Packaged | 4 .tgz chart archives |
| `bootstrap/default-workspace/charts/new-chart/` | Template | Bootstrap chart (7 files) |
| `testdata/k8s/simple-app/` | K8s Manifests | deployment, service, configmap |

### test_chart/test-chart/Chart.yaml

```yaml
apiVersion: v2
name: test-chart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

### Packaged Charts

| Archive | Purpose |
|---------|---------|
| `empty-chart-0.1.0.tgz` | Minimal chart for basic tests |
| `okteto-1.29.0-rc.2.tgz` | Integration testing (complex real-world) |
| `sourcegraph-6.1.5633.tgz` | Real-world complex chart |

### Test Files Using Fixtures

| File | Usage |
|------|-------|
| `pkg/integration/apply-changes_chart.go` | Uses okteto chart for integration tests |
| `pkg/testhelpers/pg.go` | Database setup with test data |
| `cmd/test-data.go` | Test data generation command |

---

## Implications for PR2 Implementation

### Architecture Decision Required

The PRDs assume HTTP REST between Next.js and Go. Two options:

**Option A: Add HTTP Endpoint (Recommended)**
- Add new HTTP handler to Go backend for `/api/validate`
- Next.js tool calls Go directly
- Returns results synchronously
- New pattern but simpler for tool integration

**Option B: Use Existing Queue Pattern**
- Enqueue `validate_chart` job via pg_notify
- Go worker processes asynchronously
- Results via Centrifugo WebSocket
- Consistent but complex for synchronous UX

### Tool Implementation Path

1. Add `validateChart` tool to `pkg/llm/conversational.go` following existing pattern
2. Execute `helm lint`, `helm template`, `kube-score` locally
3. Construct `tool_result` with validation findings
4. Claude interprets results on next iteration

### System Prompt Update

Add to `chatOnlySystemPrompt` in `pkg/llm/system.go`:

```go
const validateChartInstructions = `
When the user asks to validate, check, lint, or verify their chart,
use the validateChart tool to run automated validation including
helm lint, helm template, and kube-score checks.
`
```

### Test Strategy

Use existing fixtures:
- `test_chart/test-chart/` for basic validation
- Create additional fixtures for error scenarios (lint errors, kube-score warnings)

---

## Code References

### Communication
- `chartsmith-app/lib/utils/queue.ts:9-20` - Work queue enqueue
- `chartsmith/pkg/listener/start.go:13-119` - LISTEN handlers
- `chartsmith/pkg/realtime/centrifugo.go:48-66` - Real-time events

### Workspace/Chart Context
- `pkg/workspace/types/types.go:32-46` - Workspace struct
- `pkg/listener/execute-plan.go:78-80` - First chart selection

### Tools
- `pkg/llm/execute-action.go:510-531` - text_editor tool
- `pkg/llm/conversational.go:99-128` - Conversational tools
- `pkg/llm/client.go:12-21` - Anthropic SDK client

### System Prompts
- `pkg/llm/system.go` - All prompt constants
- `pkg/workspace/context.go:28-175` - File selection with embeddings

### Test Fixtures
- `test_chart/test-chart/Chart.yaml` - Basic test chart
- `testdata/charts/okteto-1.29.0-rc.2.tgz` - Integration test chart

---

*Research completed 2024-12-01*
