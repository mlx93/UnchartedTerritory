---
date: 2025-12-02T16:49:02-0600
researcher: Claude Code
git_commit: e615830f7b0b60ecb817d762b9b4e4186f855d85
branch: main
repository: UnchartedTerritory
topic: "Go Tool Calls in Anthropic SDK Format - Chartsmith Codebase"
tags: [research, codebase, anthropic-sdk, tools, chartsmith, go, workflow, custom-tags]
status: complete
last_updated: 2025-12-02
last_updated_by: Claude Code
last_updated_note: "Added follow-up research explaining how Chartsmith functions without explicit createChart/getChartContext/updateChart tools"
---

# Research: Go Tool Calls in Anthropic SDK Format - Chartsmith Codebase

**Date**: 2025-12-02T16:49:02-0600
**Researcher**: Claude Code
**Git Commit**: e615830f7b0b60ecb817d762b9b4e4186f855d85
**Branch**: main
**Repository**: UnchartedTerritory

## Research Question

What existing Go tool calls in the Anthropic SDK format currently exist in the chartsmith code folder?

## Summary

The chartsmith codebase contains **3 active tools** and **1 commented-out tool** defined in Anthropic SDK format. All tool definitions are located in the `/chartsmith/pkg/llm/` directory and use the `github.com/anthropics/anthropic-sdk-go` package. The tools follow a consistent pattern using `anthropic.ToolParam` structs with JSON Schema input definitions.

## Detailed Findings

### Active Tools

#### 1. `text_editor_20241022` (File: execute-action.go)

**Location**: `chartsmith/pkg/llm/execute-action.go:510-532`

**Purpose**: Provides file manipulation capabilities for Claude to edit Helm chart files.

**Tool Definition**:
```go
anthropic.ToolParam{
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
}
```

**Input Schema Properties**:
| Property | Type | Description |
|----------|------|-------------|
| `command` | string (enum) | One of: `view`, `str_replace`, `create` |
| `path` | string | File path to operate on |
| `old_str` | string | String to find and replace (for str_replace) |
| `new_str` | string | Replacement string or new file content |

**Handler Implementation** (lines 603-654):
- `view`: Returns file content or error if file doesn't exist
- `str_replace`: Performs string replacement in the file
- `create`: Creates new file content (errors if file exists)

**Constants Defined** (lines 19-24):
```go
const (
    TextEditor_Sonnet37 = "text_editor_20250124"
    TextEditor_Sonnet35 = "text_editor_20241022"
    Model_Sonnet37 = "claude-3-7-sonnet-20250219"
    Model_Sonnet35 = "claude-3-5-sonnet-20241022"
)
```

---

#### 2. `latest_subchart_version` (File: conversational.go)

**Location**: `chartsmith/pkg/llm/conversational.go:99-113`

**Purpose**: Retrieves the latest version of a Helm subchart by name.

**Tool Definition**:
```go
anthropic.ToolParam{
    Name:        anthropic.F("latest_subchart_version"),
    Description: anthropic.F("Return the latest version of a subchart from name"),
    InputSchema: anthropic.F(interface{}(map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "chart_name": map[string]interface{}{
                "type":        "string",
                "description": "The subchart name to get the latest version of",
            },
        },
        "required": []string{"chart_name"},
    })),
}
```

**Input Schema Properties**:
| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `chart_name` | string | Yes | The subchart name to look up |

**Handler Implementation** (lines 192-209):
- Calls `recommendations.GetLatestSubchartVersion(input.ChartName)`
- Returns version string or "?" if no ArtifactHub package found

---

#### 3. `latest_kubernetes_version` (File: conversational.go)

**Location**: `chartsmith/pkg/llm/conversational.go:114-127`

**Purpose**: Returns the latest Kubernetes version information.

**Tool Definition**:
```go
anthropic.ToolParam{
    Name:        anthropic.F("latest_kubernetes_version"),
    Description: anthropic.F("Return the latest version of Kubernetes"),
    InputSchema: anthropic.F(interface{}(map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "semver_field": map[string]interface{}{
                "type":        "string",
                "description": "One of 'major', 'minor', or 'patch'",
            },
        },
        "required": []string{"semver_description"},
    })),
}
```

**Input Schema Properties**:
| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `semver_field` | string | Yes* | One of: `major`, `minor`, `patch` |

*Note: There's a mismatch - the property is named `semver_field` but required array specifies `semver_description`.

**Handler Implementation** (lines 175-191):
- Returns hardcoded version values:
  - `"major"` → `"1"`
  - `"minor"` → `"1.32"`
  - `"patch"` → `"1.32.1"`

---

### Commented-Out Tools

#### 4. `recommended_dependency` (File: plan.go) - INACTIVE

**Location**: `chartsmith/pkg/llm/plan.go:73-88` (commented out)

**Purpose**: Was designed to recommend Helm chart dependencies (subcharts) based on requirements.

**Tool Definition (Commented)**:
```go
// anthropic.ToolParam{
//     Name:        anthropic.F("recommended_dependency"),
//     Description: anthropic.F("Recommend a specific subchart or version of a subchart given a requirement"),
//     InputSchema: anthropic.F(interface{}(map[string]interface{}{
//         "type": "object",
//         "properties": map[string]interface{}{
//             "requirement": map[string]interface{}{
//                 "type":        "string",
//                 "description": "The requirement to recommend a dependency for, e.g. Redis, Mysql",
//             },
//         },
//         "required": []string{"requirement"},
//     })),
// }
```

**Status**: This tool definition is commented out on lines 73-88, and the `Tools` parameter in the API call is also commented out on line 93.

---

## Architecture Documentation

### SDK Integration Pattern

**Import**:
```go
import anthropic "github.com/anthropics/anthropic-sdk-go"
```

**Client Initialization** (`client.go:12-21`):
```go
func newAnthropicClient(ctx context.Context) (*anthropic.Client, error) {
    if param.Get().AnthropicAPIKey == "" {
        return nil, fmt.Errorf("ANTHROPIC_API_KEY is not set")
    }
    return anthropic.NewClient(
        option.WithAPIKey(param.Get().AnthropicAPIKey),
    ), nil
}
```

### Tool Definition Pattern

1. **Define tools as `[]anthropic.ToolParam`**
2. **Convert to `[]anthropic.ToolUnionUnionParam`** before API call:
```go
toolUnionParams := make([]anthropic.ToolUnionUnionParam, len(tools))
for i, tool := range tools {
    toolUnionParams[i] = tool
}
```

3. **Pass to streaming API**:
```go
stream := client.Messages.NewStreaming(ctx, anthropic.MessageNewParams{
    Model:     anthropic.F(anthropic.ModelClaude3_7Sonnet20250219),
    MaxTokens: anthropic.F(int64(8192)),
    Messages:  anthropic.F(messages),
    Tools:     anthropic.F(toolUnionParams),
})
```

### Tool Execution Pattern (Agentic Loop)

1. **Detect tool calls** by checking `ContentBlockTypeToolUse`
2. **Parse tool input** from JSON
3. **Execute handler** based on tool name
4. **Return result** using `anthropic.NewToolResultBlock(block.ID, result, false)`
5. **Append to messages** as user role and continue loop

## Code References

| File | Location | Description |
|------|----------|-------------|
| `chartsmith/pkg/llm/client.go:12-21` | Client initialization with API key |
| `chartsmith/pkg/llm/execute-action.go:19-24` | Tool/model constants |
| `chartsmith/pkg/llm/execute-action.go:510-532` | `text_editor_20241022` tool definition |
| `chartsmith/pkg/llm/execute-action.go:603-654` | Text editor command handlers |
| `chartsmith/pkg/llm/conversational.go:99-113` | `latest_subchart_version` tool definition |
| `chartsmith/pkg/llm/conversational.go:114-127` | `latest_kubernetes_version` tool definition |
| `chartsmith/pkg/llm/conversational.go:175-209` | Conversational tool handlers |
| `chartsmith/pkg/llm/plan.go:73-88` | `recommended_dependency` (commented out) |

## Quick Reference Summary

| Tool Name | File | Status | Purpose |
|-----------|------|--------|---------|
| `text_editor_20241022` | execute-action.go | Active | File editing (view/replace/create) |
| `latest_subchart_version` | conversational.go | Active | Subchart version lookup |
| `latest_kubernetes_version` | conversational.go | Active | K8s version info |
| `recommended_dependency` | plan.go | Commented Out | Dependency recommendations |

## Open Questions

1. Why is `recommended_dependency` commented out? Was this functionality deprecated or moved elsewhere?
2. The `latest_kubernetes_version` tool has a schema mismatch (`semver_field` vs `semver_description` in required array) - is this intentional?
3. The `text_editor_20250124` constant exists but isn't used - is there a plan to migrate to this newer version?

---

## Follow-up Research: 2025-12-02T17:15:00-0600

### Research Question

If these are the only tools that exist, how does Chartsmith function for core operations like createChart, getChartContext, and updateChart?

### Key Finding: Chartsmith Does NOT Use Tools for Core Chart Operations

**Chartsmith uses a fundamentally different architecture** than explicit tool calling for its core chart manipulation operations. Instead of defining `createChart`, `getChartContext`, `updateChart` as Anthropic tools, it uses:

1. **Custom XML-like tags** parsed from raw LLM text responses
2. **System prompts** that instruct LLMs to return structured output in these tags
3. **Queue-based workflow orchestration** via PostgreSQL LISTEN/NOTIFY
4. **Vector embeddings** for context/file selection
5. **The `text_editor` tool only for final file editing** (view/str_replace/create commands)

---

### How Chart Creation Works

**Location**: `chartsmith/pkg/listener/new-conversion.go` → `chartsmith/pkg/llm/convert-file.go`

**Process**:
1. User uploads Kubernetes manifests
2. Files are sorted by GVK priority (ConfigMap=0, Secret=1, Deployment=8, etc.)
3. Each file is converted one-by-one via LLM (Groq Llama 3.3 70B or Claude)
4. LLM returns `<chartsmithArtifact path="...">content</chartsmithArtifact>` tags
5. Parser extracts file paths and content from tags (`chartsmith/pkg/llm/parser.go:31-89`)
6. For `values.yaml`, unified diff patches are applied to merge values incrementally
7. All files stored in database, chart record created

**No tool call involved** - Pure text generation with custom tag parsing.

---

### How Context Retrieval Works

**Location**: `chartsmith/pkg/workspace/context.go:28-175`

**Function**: `ChooseRelevantFilesForChatMessage()`

**Process**:
1. User prompt is **expanded** by Claude to include semantic search terms (`chartsmith/pkg/llm/expand.go:10-54`)
2. Embeddings generated via **Voyage AI** (`chartsmith/pkg/embedding/embeddings.go:35-116`)
3. **Vector similarity search** using pgvector: `1 - (embeddings <=> $1)` (cosine distance)
4. `Chart.yaml` and `values.yaml` always included with score 1.0
5. Non-YAML files penalized by -0.25 similarity
6. Top 10 files with similarity >= 0.8 selected
7. File contents included in LLM prompt context

**No tool call involved** - Database queries and vector search, then files embedded in system prompts.

---

### How Chart Updates Work

**Location**: `chartsmith/pkg/llm/execute-action.go:437-676`

**Process**:
1. User approves a plan (text description of changes)
2. **Execution plan created** - LLM returns `<chartsmithActionPlan>` tags listing file actions
3. Action files stored in database (path + action: "create" or "update")
4. **For each file**, `ExecuteAction()` is called with the `text_editor_20241022` tool
5. Claude uses tool iteratively:
   - `view` to see current content
   - `str_replace` to make changes (with fuzzy matching fallback)
   - `create` to make new files
6. Content streamed to UI via realtime events

**This is the ONLY place tools are used** - and only for the actual file editing step, not for planning or context.

---

### Architecture Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CHARTSMITH WORKFLOW                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. PLAN CREATION (No Tools)                                          │
│     ├─ User sends chat message                                        │
│     ├─ Context selected via vector similarity search                  │
│     ├─ LLM streams plain text plan description                        │
│     └─ Plan stored in database                                        │
│                                                                       │
│  2. EXECUTION PLAN (Custom Tags, No Tools)                            │
│     ├─ User approves plan                                             │
│     ├─ LLM returns <chartsmithActionPlan> tags                        │
│     ├─ Parser.ParsePlan() extracts action items                       │
│     └─ Action files stored in database                                │
│                                                                       │
│  3. ACTION EXECUTION (text_editor Tool)                               │
│     ├─ For each action file                                           │
│     ├─ Claude uses text_editor_20241022 tool                          │
│     │   ├─ view: read current content                                 │
│     │   ├─ str_replace: modify content                                │
│     │   └─ create: create new file                                    │
│     └─ Changes saved to database                                      │
│                                                                       │
│  4. CONVERSION (Custom Tags, No Tools)                                │
│     ├─ K8s manifests uploaded                                         │
│     ├─ LLM returns <chartsmithArtifact> tags                          │
│     ├─ Parser.ParseArtifacts() extracts file contents                 │
│     ├─ Unified diffs applied for values.yaml                          │
│     └─ Chart files stored in database                                 │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

### Custom Tag Formats Used

| Tag | Purpose | Location |
|-----|---------|----------|
| `<chartsmithArtifact path="...">content</chartsmithArtifact>` | File content output | `parser.go:35` |
| `<chartsmithArtifactPlan title="...">` | Plan container | `parser.go:96` |
| `<chartsmithActionPlan type="file" action="create/update" path="...">` | Action items | `parser.go:103` |

**Parsing regex patterns**:
- Artifacts: `(?s)<chartsmithArtifact([^>]*)>(.*?)</chartsmithArtifact>`
- Actions: `<chartsmithActionPlan\s+type="([^"]+)"\s+action="([^"]+)"\s+path="([^"]+)"`

---

### LLM Models by Operation

| Operation | Model | Provider | Tools Used |
|-----------|-------|----------|------------|
| Intent classification | llama-3.3-70b-versatile | Groq | None (JSON response format) |
| Prompt expansion | claude-3-7-sonnet-20250219 | Anthropic | None |
| Plan creation | claude-3-7-sonnet-20250219 | Anthropic | None |
| Execution plan | claude-3-7-sonnet-20250219 | Anthropic | None |
| File conversion | llama-3.3-70b-versatile | Groq | None |
| Action execution | claude-3-5-sonnet-20241022 | Anthropic | `text_editor_20241022` |
| Conversational chat | claude-3-7-sonnet-20250219 | Anthropic | `latest_subchart_version`, `latest_kubernetes_version` |

---

### Key Code References

| Operation | File | Lines | Description |
|-----------|------|-------|-------------|
| Chart creation entry | `listener/new-conversion.go` | 23-130 | Conversion workflow start |
| File conversion | `llm/convert-file.go` | 24-121 | LLM call for K8s→Helm |
| Artifact parsing | `llm/parser.go` | 31-89 | Extract `<chartsmithArtifact>` tags |
| Action parsing | `llm/parser.go` | 91-135 | Extract `<chartsmithActionPlan>` tags |
| Context selection | `workspace/context.go` | 28-175 | Vector similarity file selection |
| Prompt expansion | `llm/expand.go` | 10-54 | Semantic search term generation |
| Plan creation | `llm/plan.go` | 21-116 | Plain text plan streaming |
| Execution plan | `llm/execute-plan.go` | 14-105 | Action plan generation |
| Action execution | `llm/execute-action.go` | 437-676 | text_editor tool loop |
| File updates | `workspace/file.go` | 107-165 | Database persistence |

---

### Why No Explicit Chart Tools?

The architecture reflects these design decisions:

1. **Separation of Concerns**: Planning is text, execution uses tools
2. **Flexibility**: Custom tags allow arbitrary structured output without predefined schemas
3. **Streaming**: Plans can stream to UI before being complete
4. **Stateful Context**: Files selected via embeddings, not tool calls
5. **Multi-Model Support**: Groq (Llama) and Anthropic (Claude) use same tag format

This means any **AI SDK migration** would need to:
- Keep the custom tag parsing infrastructure
- Only convert the `text_editor_20241022` tool to new SDK format
- Consider if `latest_subchart_version`/`latest_kubernetes_version` should become SDK tools
