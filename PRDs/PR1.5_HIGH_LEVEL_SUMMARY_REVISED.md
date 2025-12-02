# PR1.5 High-Level Summary (Revised)

**Name**: Migration & Feature Parity  
**Timeline**: Days 3-4  
**Prerequisite**: PR1 merged  
**Goal**: Complete AI SDK migration with **full feature parity**; Go becomes pure application logic  
**Updated**: December 2, 2025

---

## Key Architecture Decision

**Go does NOT need to make LLM calls after PR1.5.**

All "thinking" (chat, tool orchestration, reasoning) lives in AI SDK Core (Node).  
Go becomes a pure service layer for application logic (chart CRUD, file ops, validation).

**After PR1.5, no user-facing chat UI paths route through the legacy Go LLM stack (`pkg/llm/*`). The only chat interface available in the product is the AI SDK-powered `/api/chat` route.**

This simplifies PR1.5 significantly:
- ❌ No AI Gateway endpoint needed
- ❌ No Go LLM client modifications needed  
- ✅ Go just exposes HTTP endpoints for tool execution
- ✅ Cleaner separation of concerns
- ✅ **All existing tools ported for full feature parity**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     ARCHITECTURE AFTER PR1.5                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────┐     useChat      ┌─────────────────┐               │
│  │   Chat UI       │ ────────────────►│  /api/chat      │               │
│  │  (AI SDK)       │ ◄────────────────│  streamText()   │               │
│  └─────────────────┘   Data Stream    │  + 6 tools      │               │
│                                       └────────┬────────┘               │
│                                                │                         │
│         ALL LLM "THINKING" HAPPENS HERE        │ tool.execute()         │
│         (Node / AI SDK Core)                   │                         │
│                                                ▼                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      GO BACKEND                                  │    │
│  │                  (Pure Application Logic)                        │    │
│  │                                                                  │    │
│  │   POST /api/tools/charts/create       →  Create chart in DB     │    │
│  │   GET  /api/tools/charts/:id          →  Load chart context     │    │
│  │   PATCH /api/tools/charts/:id         →  Update chart           │    │
│  │   POST /api/tools/editor              →  File operations        │    │
│  │   POST /api/tools/versions/subchart   →  ArtifactHub lookup     │    │
│  │   POST /api/tools/versions/kubernetes →  K8s version info       │    │
│  │                                                                  │    │
│  │   NO LLM CALLS - Just executes requested operations             │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │
│  LEGACY (Deprecated - no longer used for user chat)                     │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │
│  pkg/llm/* - Mark as legacy, document for future removal                │
│  pkg/listener/chat handlers - No longer triggered by user chat          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Deliverables (Ordered by Priority)

### MUST HAVE (Core Migration)

#### 1. Shared LLM Client Library
**File**: `lib/ai/llmClient.ts`

```typescript
// Single source of truth for all LLM configuration
export async function runChat(options: {
  model: string;
  messages: Message[];
  system?: string;
  tools?: ToolSet;
}) {
  return streamText({
    model: getModel(options.model),
    messages: options.messages,
    system: options.system,
    tools: options.tools,
  });
}

export function getModel(modelId: string) {
  // Provider factory - returns OpenRouter model
}

export const AVAILABLE_MODELS = [
  { id: 'openai/gpt-4o', name: 'GPT-4o', provider: 'openai' },
  { id: 'anthropic/claude-3.5-sonnet', name: 'Claude 3.5 Sonnet', provider: 'anthropic' },
];
```

**Why**: Centralizes provider config, model selection, and common options in one place.

---

#### 2. Go HTTP Server
**File**: `pkg/api/server.go`

```go
func StartHTTPServer(port string) {
    mux := http.NewServeMux()
    
    // Chart endpoints
    mux.HandleFunc("POST /api/tools/charts/create", handlers.CreateChart)
    mux.HandleFunc("GET /api/tools/charts/{id}", handlers.GetChartContext)
    mux.HandleFunc("PATCH /api/tools/charts/{id}", handlers.UpdateChart)
    
    // Editor endpoint
    mux.HandleFunc("POST /api/tools/editor", handlers.TextEditor)
    
    // Version lookup endpoints (feature parity)
    mux.HandleFunc("POST /api/tools/versions/subchart", handlers.GetSubchartVersion)
    mux.HandleFunc("POST /api/tools/versions/kubernetes", handlers.GetKubernetesVersion)
    
    http.ListenAndServe(":"+port, mux)
}
```

**Why**: Provides HTTP interface for AI SDK tools to call.

---

#### 3. createChart Tool + Endpoint

**Tool Definition**: `lib/ai/tools/createChart.ts`

```typescript
import { tool } from 'ai';
import { z } from 'zod';

export const createChart = tool({
  description: `Create a new Helm chart based on user requirements. 
    Use this when the user wants to create a new chart, start a new project,
    or when currentRevisionNumber is 0 and user expresses creation intent.`,
  
  parameters: z.object({
    name: z.string().describe('Chart name (e.g., "nginx", "my-app")'),
    description: z.string().describe('What the chart should do'),
    type: z.enum(['deployment', 'statefulset', 'cronjob', 'custom'])
      .optional()
      .describe('Primary workload type'),
    includeIngress: z.boolean().optional().describe('Include ingress resource'),
    includeService: z.boolean().optional().describe('Include service resource'),
  }),

  execute: async (params) => {
    const response = await fetch(`${process.env.GO_BACKEND_URL}/api/tools/charts/create`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(params),
    });
    
    if (!response.ok) {
      throw new Error(`Chart creation failed: ${response.statusText}`);
    }
    
    return response.json();
  },
});
```

**Go Handler**: `pkg/api/handlers/charts.go`

```go
type CreateChartRequest struct {
    Name           string `json:"name"`
    Description    string `json:"description"`
    Type           string `json:"type,omitempty"`
    IncludeIngress bool   `json:"includeIngress,omitempty"`
    IncludeService bool   `json:"includeService,omitempty"`
}

type CreateChartResponse struct {
    Success     bool   `json:"success"`
    ChartID     string `json:"chartId"`
    WorkspaceID string `json:"workspaceId"`
    Message     string `json:"message"`
    Files       []struct {
        Path    string `json:"path"`
        Content string `json:"content"`
    } `json:"files"`
}

func CreateChart(w http.ResponseWriter, r *http.Request) {
    var req CreateChartRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // 1. Create workspace in DB
    // 2. Generate initial chart files (Chart.yaml, values.yaml, templates/)
    // 3. Create initial revision
    // 4. Trigger pg_notify for any listeners
    // 5. Return response with chart details
    
    resp := CreateChartResponse{
        Success: true,
        ChartID: chartID,
        // ...
    }
    
    json.NewEncoder(w).Encode(resp)
}
```

**Why**: Core demo requirement - "Demonstrate creating a new chart via chat"

---

#### 4. Register Tools in /api/chat

**File**: `app/api/chat/route.ts` (modify from PR1)

```typescript
import { runChat } from '@/lib/ai/llmClient';
import { createChart } from '@/lib/ai/tools/createChart';
import { getChartContext } from '@/lib/ai/tools/getChartContext';
import { updateChart } from '@/lib/ai/tools/updateChart';
import { textEditor } from '@/lib/ai/tools/textEditor';
import { latestSubchartVersion } from '@/lib/ai/tools/latestSubchartVersion';
import { latestKubernetesVersion } from '@/lib/ai/tools/latestKubernetesVersion';
import { getSystemPrompt } from '@/lib/ai/prompts';

export async function POST(req: Request) {
  const { messages, model, workspaceId, revisionNumber } = await req.json();
  
  const result = await runChat({
    model,
    messages,
    system: getSystemPrompt({ workspaceId, revisionNumber }),
    tools: {
      createChart,
      getChartContext,
      updateChart,
      textEditor,
      latestSubchartVersion,    // Feature parity - existing tool
      latestKubernetesVersion,  // Feature parity - existing tool
    },
  });

  return result.toDataStreamResponse();
}
```

---

#### 5. System Prompt Migration

**File**: `lib/ai/prompts.ts`

```typescript
export function getSystemPrompt(context: {
  workspaceId?: string;
  revisionNumber?: number;
  chartContext?: string;
}) {
  const basePrompt = `You are Chartsmith, an AI assistant specialized in creating 
and managing Helm charts for Kubernetes deployments.

## Available Tools
- createChart: Create a new Helm chart from requirements
- getChartContext: Load current chart files and metadata  
- updateChart: Modify existing chart configuration
- textEditor: View, edit, or create specific files
- latestSubchartVersion: Look up latest subchart versions on ArtifactHub
- latestKubernetesVersion: Get current Kubernetes version information

## Behavior Guidelines
- When revisionNumber is 0 or user wants a new chart, use createChart
- For questions about current chart state, use getChartContext first
- For specific file changes, use textEditor
- For subchart dependencies, use latestSubchartVersion to find current versions
- For K8s API compatibility questions, use latestKubernetesVersion
- Always explain what you're doing and why`;

  // Add chart context if available
  if (context.chartContext) {
    return `${basePrompt}\n\n## Current Chart Context\n${context.chartContext}`;
  }
  
  return basePrompt;
}
```

**Source**: Adapt from `pkg/llm/system.go` (copy relevant prompt content)

---

#### 6. Remove @anthropic-ai/sdk from TypeScript

**Actions**:
- Remove from `package.json`
- Delete `lib/llm/prompt-type.ts` (orphaned code)
- Search for any remaining imports, remove them
- Verify: `grep -r "anthropic" chartsmith-app/` returns nothing

---

#### 7. Integration Test

**File**: `tests/integration/create-chart.test.ts`

```typescript
describe('Create Chart via AI SDK Chat', () => {
  it('should create a new chart when user requests', async () => {
    const response = await fetch('/api/chat', {
      method: 'POST',
      body: JSON.stringify({
        messages: [{ role: 'user', content: 'Create a basic nginx deployment chart' }],
        model: 'openai/gpt-4o',
        revisionNumber: 0,
      }),
    });
    
    // Assert tool was called
    const result = await response.json();
    expect(result.toolCalls).toContainEqual(
      expect.objectContaining({ name: 'createChart' })
    );
    
    // Assert chart was created (check DB or response)
    // ...
  });
});
```

---

### NICE TO HAVE (Complete Feature Parity)

#### 8. getChartContext Tool + Endpoint

**Tool**: `lib/ai/tools/getChartContext.ts`

```typescript
export const getChartContext = tool({
  description: `Load the current chart's files and metadata. Use this when:
    - User asks about the current chart state
    - You need to understand the chart before making changes
    - User asks "what does this chart do?"`,
  
  parameters: z.object({
    workspaceId: z.string().describe('Workspace ID'),
    includeFiles: z.boolean().optional().describe('Include file contents'),
  }),

  execute: async (params) => {
    const response = await fetch(
      `${process.env.GO_BACKEND_URL}/api/tools/charts/${params.workspaceId}`,
      { method: 'GET' }
    );
    return response.json();
  },
});
```

**Go Handler**: `GET /api/tools/charts/{id}`

```go
type GetChartContextResponse struct {
    WorkspaceID string `json:"workspaceId"`
    Name        string `json:"name"`
    Description string `json:"description"`
    Revision    int    `json:"revision"`
    Files       []struct {
        Path    string `json:"path"`
        Content string `json:"content"`
    } `json:"files"`
    Values map[string]interface{} `json:"values"`
}
```

---

#### 9. updateChart Tool + Endpoint

**Tool**: `lib/ai/tools/updateChart.ts`

```typescript
export const updateChart = tool({
  description: `Update an existing chart's configuration. Use this for:
    - Changing values (replicas, image tags, etc.)
    - Adding/removing resources
    - Modifying chart metadata`,
  
  parameters: z.object({
    workspaceId: z.string(),
    changes: z.array(z.object({
      type: z.enum(['setValue', 'addResource', 'removeResource', 'updateMetadata']),
      path: z.string().optional(),
      value: z.any().optional(),
    })),
    message: z.string().describe('Commit message for this change'),
  }),

  execute: async (params) => {
    const response = await fetch(
      `${process.env.GO_BACKEND_URL}/api/tools/charts/${params.workspaceId}`,
      {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(params),
      }
    );
    return response.json();
  },
});
```

---

#### 10. textEditor Tool + Endpoint

**Tool**: `lib/ai/tools/textEditor.ts`

```typescript
export const textEditor = tool({
  description: `View, edit, or create files in the chart. Use this for:
    - Viewing specific file contents
    - Making precise text changes
    - Creating new template files`,
  
  parameters: z.object({
    command: z.enum(['view', 'create', 'str_replace', 'insert']),
    workspaceId: z.string(),
    path: z.string().describe('File path relative to chart root'),
    content: z.string().optional().describe('For create: full content'),
    oldStr: z.string().optional().describe('For str_replace: text to find'),
    newStr: z.string().optional().describe('For str_replace: replacement text'),
    insertLine: z.number().optional().describe('For insert: line number'),
    insertText: z.string().optional().describe('For insert: text to insert'),
  }),

  execute: async (params) => {
    const response = await fetch(`${process.env.GO_BACKEND_URL}/api/tools/editor`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(params),
    });
    return response.json();
  },
});
```

---

#### 11. Deprecation Banner & Documentation

**ChatContainer.tsx** (legacy component):
```tsx
// Add deprecation notice
<Banner variant="warning">
  This chat interface is deprecated. Please use the new AI-powered chat.
</Banner>
```

**ARCHITECTURE.md** updates:
- Document new tool → Go HTTP pattern
- Explain that Go no longer makes LLM calls
- Mark `pkg/llm/*` as legacy

---

## Files Summary

### New Files (Next.js) - 10 files
```
lib/ai/
├── llmClient.ts                    # Shared LLM client (MUST HAVE)
├── prompts.ts                      # System prompts (MUST HAVE)
└── tools/
    ├── createChart.ts              # Chart creation (MUST HAVE)
    ├── latestSubchartVersion.ts    # Subchart lookup (MUST HAVE - feature parity)
    ├── latestKubernetesVersion.ts  # K8s version (MUST HAVE - feature parity)
    ├── textEditor.ts               # File ops (MUST HAVE - feature parity)
    ├── utils.ts                    # Shared error handling (MUST HAVE)
    ├── getChartContext.ts          # Chart loading (NICE TO HAVE - new tool)
    └── updateChart.ts              # Chart updates (NICE TO HAVE - new tool)

tests/integration/
└── create-chart.test.ts            # Integration test (MUST HAVE)
```

### New Files (Go) - 5 files
```
pkg/api/
├── server.go                       # HTTP server (MUST HAVE)
├── errors.go                       # Error response utilities (MUST HAVE)
└── handlers/
    ├── charts.go                   # Chart endpoints (MUST HAVE)
    ├── versions.go                 # Version lookup endpoints (MUST HAVE)
    └── editor.go                   # Editor endpoint (MUST HAVE - feature parity)
```

### Modified Files - 4 files
```
app/api/chat/route.ts               # Add 6 tools (MUST HAVE)
cmd/run.go                          # Start HTTP server (MUST HAVE)
package.json                        # Remove @anthropic-ai/sdk (MUST HAVE)
ARCHITECTURE.md                     # Documentation (NICE TO HAVE)
```

### Files to Mark as Legacy (No Changes Required)
```
pkg/llm/client.go                   # Document as legacy
pkg/llm/conversational.go           # Document as legacy
pkg/llm/execute-action.go           # Document as legacy
pkg/llm/system.go                   # Source for prompt migration
```

---

## What This Removes vs. Original Plan

| Original Plan | Revised Plan | Reason |
|---------------|--------------|--------|
| `pkg/llm/gateway_client.go` | ❌ Removed | Go doesn't need LLM calls |
| Modify `pkg/llm/client.go` | ❌ Removed | Just mark as legacy |
| Modify `pkg/llm/execute-action.go` | ❌ Removed | Just mark as legacy |
| `/api/ai/llm` gateway endpoint | ❌ Removed | Go doesn't need to call it |
| Complex Go LLM refactoring | ❌ Removed | Go becomes pure app logic |

**Net result**: ~40% less Go modification work, cleaner architecture

---

## Success Criteria

### MUST PASS
- [ ] "Create a nginx chart" works end-to-end via AI SDK chat
- [ ] AI SDK chat must invoke createChart for a new-chart conversation, and Go must create a valid workspace (validated via integration test)
- [ ] createChart tool invokes Go endpoint correctly
- [ ] Chart appears in database after creation
- [ ] **latestSubchartVersion tool returns valid version data** (feature parity)
- [ ] **latestKubernetesVersion tool returns valid version data** (feature parity)
- [ ] Streaming responses work (from PR1)
- [ ] No @anthropic-ai/sdk in node_modules
- [ ] Integration test passes
- [ ] **Error responses follow standard format** (success, message, optional code/details)

### QUALITY GATES
- [ ] All **4 feature-parity tools** implemented as MUST HAVE (createChart, textEditor, latestSubchartVersion, latestKubernetesVersion)
- [ ] All **6 tools** implemented for full quality (adds getChartContext, updateChart as NICE TO HAVE)
- [ ] Clear endpoint naming (`/api/tools/charts/create`, `/api/tools/versions/subchart`, etc.)
- [ ] System prompts document all 6 available tools
- [ ] **Error handling utility (`callGoEndpoint`) shared across all tools**
- [ ] **All Go tool endpoints enforce existing workspace ownership and authentication middleware**
- [ ] ARCHITECTURE.md updated
- [ ] Legacy code marked appropriately

---

## Environment Variables

```env
# From PR1
OPENROUTER_API_KEY=sk-or-v1-xxxxx
DEFAULT_AI_PROVIDER=openai
DEFAULT_AI_MODEL=openai/gpt-4o

# New in PR1.5
GO_BACKEND_URL=http://localhost:8080
```

---

## What PR1.5 Enables for PR2

After PR1.5 is complete, PR2 can:
- Add `validateChart` tool to existing tools array (just another tool)
- Add `/api/validate` endpoint to existing Go HTTP server (just another handler)
- Reuse the proven `tool.execute() → Go HTTP → response` pattern
- Focus purely on validation logic (helm lint, kube-score), not infrastructure

---

## Risk Mitigation

### If Time Runs Short

**Priority order**:
1. `createChart` + Go endpoint (demo requirement)
2. `llmClient.ts` + tool registration
3. System prompts
4. Integration test
5. **`latestSubchartVersion`** (feature parity - REQUIRED)
6. **`latestKubernetesVersion`** (feature parity - REQUIRED)
7. **`textEditor`** (feature parity - REQUIRED, existing tool)
8. **Error response contract** (production quality)
9. `getChartContext` (new tool - NICE TO HAVE)
10. `updateChart` (new tool - NICE TO HAVE)
11. Deprecation banner/docs

**Absolute minimum viable**: Items 1-8 complete for full feature parity

**Existing tools that MUST be ported** (3 total):
- `text_editor_20241022` → `textEditor`
- `latest_subchart_version` → `latestSubchartVersion`
- `latest_kubernetes_version` → `latestKubernetesVersion`

**Note on feature parity**: The original requirement states "All existing features continue to work (tool calling, file context, etc.)". The utility tools are existing features that must be ported.

### If Go HTTP Server Is Complex

- Start with just `POST /api/tools/charts/create`
- Other endpoints can be stubs returning mock data
- Prove the pattern works, fill in implementation
- See `PRDs/PR1.5 Go Handlers Imp Notes.md` for detailed implementation guidance

---

## Security Note

All PR1.5 Go HTTP tool endpoints (`/api/tools/...`) will enforce existing workspace ownership and authentication middleware. Only the owner or authorized collaborators may modify or read workspace resources.

---

*End of Document*
