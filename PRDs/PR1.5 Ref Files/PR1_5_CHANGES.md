# PR1.5 Changes Document

**Purpose**: Amendments to PR1.5 plan based on requirements review  
**Date**: December 2, 2025  
**Status**: Ready for incorporation

---

## Change 1: Add Utility Tool Ports

### Rationale

Original requirement states: *"All existing features continue to work (tool calling, file context, etc.)"*

The two utility tools must be ported to maintain full feature parity.

### New MUST HAVE Tasks

#### Task 8: latestSubchartVersion Tool + Endpoint

**Files**: `lib/ai/tools/latestSubchartVersion.ts`, `pkg/api/handlers/versions.go`

**Tool Definition**:
```typescript
import { tool } from 'ai';
import { z } from 'zod';

export const latestSubchartVersion = tool({
  description: `Look up the latest version of a Helm subchart on ArtifactHub. 
    Use when user asks about subchart versions, dependencies, or wants to 
    add a subchart to their chart.`,

  parameters: z.object({
    chartName: z.string().describe('Name of the subchart (e.g., "postgresql", "redis")'),
    repository: z.string().optional().describe('Specific repo (defaults to ArtifactHub search)'),
  }),

  execute: async (params) => {
    const response = await fetch(`${process.env.GO_BACKEND_URL}/api/tools/versions/subchart`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(params),
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.error || 'Subchart lookup failed');
    }
    
    return response.json();
  },
});
```

**Go Handler**: `POST /api/tools/versions/subchart`

```go
type SubchartVersionRequest struct {
    ChartName  string `json:"chartName"`
    Repository string `json:"repository,omitempty"`
}

type SubchartVersionResponse struct {
    Success    bool   `json:"success"`
    ChartName  string `json:"chartName"`
    Version    string `json:"version"`
    AppVersion string `json:"appVersion,omitempty"`
    Repository string `json:"repository"`
    URL        string `json:"url,omitempty"`
}

func GetSubchartVersion(w http.ResponseWriter, r *http.Request) {
    var req SubchartVersionRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // Query ArtifactHub API for latest version
    // Reuse existing logic from pkg/llm/conversational.go
    
    resp := SubchartVersionResponse{
        Success:   true,
        ChartName: req.ChartName,
        Version:   latestVersion,
        // ...
    }
    
    json.NewEncoder(w).Encode(resp)
}
```

**Implementation Note**: Port logic from existing `latest_subchart_version` tool in `pkg/llm/conversational.go`. This is an external API call to ArtifactHub, no database interaction required.

---

#### Task 9: latestKubernetesVersion Tool + Endpoint

**Files**: `lib/ai/tools/latestKubernetesVersion.ts`, `pkg/api/handlers/versions.go`

**Tool Definition**:
```typescript
import { tool } from 'ai';
import { z } from 'zod';

export const latestKubernetesVersion = tool({
  description: `Get current Kubernetes version information. Use when user asks 
    about K8s versions, API compatibility, or which version to target.`,

  parameters: z.object({
    channel: z.enum(['stable', 'latest', 'all']).optional()
      .describe('Version channel (defaults to stable)'),
  }),

  execute: async (params) => {
    const response = await fetch(`${process.env.GO_BACKEND_URL}/api/tools/versions/kubernetes`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(params),
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.error || 'Kubernetes version lookup failed');
    }
    
    return response.json();
  },
});
```

**Go Handler**: `POST /api/tools/versions/kubernetes`

```go
type KubernetesVersionRequest struct {
    Channel string `json:"channel,omitempty"` // "stable", "latest", "all"
}

type KubernetesVersionResponse struct {
    Success        bool     `json:"success"`
    StableVersion  string   `json:"stableVersion"`
    LatestVersion  string   `json:"latestVersion,omitempty"`
    SupportedRange []string `json:"supportedRange,omitempty"`
    EOLDate        string   `json:"eolDate,omitempty"`
}

func GetKubernetesVersion(w http.ResponseWriter, r *http.Request) {
    var req KubernetesVersionRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // Fetch from kubernetes version API or cached data
    // Reuse existing logic from pkg/llm/conversational.go
    
    resp := KubernetesVersionResponse{
        Success:       true,
        StableVersion: "1.31",
        // ...
    }
    
    json.NewEncoder(w).Encode(resp)
}
```

**Implementation Note**: Port logic from existing `latest_kubernetes_version` tool in `pkg/llm/conversational.go`. May query official K8s release API or use cached/hardcoded data.

---

### Updated File Summary

#### New Files (Next.js) - Add 2 files

| File | Priority |
|------|----------|
| `lib/ai/tools/latestSubchartVersion.ts` | MUST |
| `lib/ai/tools/latestKubernetesVersion.ts` | MUST |

#### New Files (Go) - Add 1 file

| File | Priority |
|------|----------|
| `pkg/api/handlers/versions.go` | MUST |

#### Updated Route Registration

**File**: `pkg/api/server.go`

Add routes:
```go
mux.HandleFunc("POST /api/tools/versions/subchart", handlers.GetSubchartVersion)
mux.HandleFunc("POST /api/tools/versions/kubernetes", handlers.GetKubernetesVersion)
```

#### Updated Tool Registration

**File**: `app/api/chat/route.ts`

```typescript
import { latestSubchartVersion } from '@/lib/ai/tools/latestSubchartVersion';
import { latestKubernetesVersion } from '@/lib/ai/tools/latestKubernetesVersion';

// In runChat tools object:
tools: {
  createChart,
  getChartContext,
  updateChart,
  textEditor,
  latestSubchartVersion,    // NEW
  latestKubernetesVersion,  // NEW
},
```

#### Updated System Prompt

**File**: `lib/ai/prompts.ts`

Add to Available Tools section:
```typescript
## Available Tools
- createChart: Create a new Helm chart from requirements
- getChartContext: Load current chart files and metadata  
- updateChart: Modify existing chart configuration
- textEditor: View, edit, or create specific files
- latestSubchartVersion: Look up latest subchart versions on ArtifactHub
- latestKubernetesVersion: Get current Kubernetes version information
```

---

### Updated Task Priority Order

1. createChart + Go endpoint
2. llmClient.ts + tool registration
3. System prompts
4. Integration test
5. getChartContext
6. textEditor
7. updateChart
8. **latestSubchartVersion** ← NEW (moved from deferred)
9. **latestKubernetesVersion** ← NEW (moved from deferred)
10. Documentation

**Minimum viable**: Tasks 1-4 + 8-9 for full feature parity on tools

---

## Change 2: Error Response Contract

### Standard Error Response Format

All Go HTTP handlers return errors in a consistent format:

```go
// pkg/api/errors.go

type ErrorResponse struct {
    Success bool   `json:"success"`
    Error   string `json:"error"`
    Code    string `json:"code"`
    Details any    `json:"details,omitempty"`
}

// Error codes
const (
    ErrCodeValidation    = "VALIDATION_ERROR"
    ErrCodeNotFound      = "NOT_FOUND"
    ErrCodeDatabase      = "DATABASE_ERROR"
    ErrCodeExternal      = "EXTERNAL_API_ERROR"
    ErrCodeInternal      = "INTERNAL_ERROR"
    ErrCodeUnauthorized  = "UNAUTHORIZED"
)

func WriteError(w http.ResponseWriter, statusCode int, code string, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(ErrorResponse{
        Success: false,
        Error:   message,
        Code:    code,
    })
}

func WriteErrorWithDetails(w http.ResponseWriter, statusCode int, code string, message string, details any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(ErrorResponse{
        Success: false,
        Error:   message,
        Code:    code,
        Details: details,
    })
}
```

### HTTP Status Code Mapping

| Error Code | HTTP Status | When Used |
|------------|-------------|-----------|
| `VALIDATION_ERROR` | 400 | Invalid request parameters |
| `NOT_FOUND` | 404 | Workspace/chart doesn't exist |
| `DATABASE_ERROR` | 500 | DB connection or query failure |
| `EXTERNAL_API_ERROR` | 502 | ArtifactHub/K8s API failure |
| `INTERNAL_ERROR` | 500 | Unexpected server error |
| `UNAUTHORIZED` | 401 | Missing/invalid auth (if applicable) |

### Error Response Examples

**Validation Error** (400):
```json
{
  "success": false,
  "error": "Chart name is required",
  "code": "VALIDATION_ERROR",
  "details": {
    "field": "name",
    "constraint": "required"
  }
}
```

**Not Found** (404):
```json
{
  "success": false,
  "error": "Workspace not found",
  "code": "NOT_FOUND"
}
```

**External API Error** (502):
```json
{
  "success": false,
  "error": "Failed to fetch subchart version from ArtifactHub",
  "code": "EXTERNAL_API_ERROR",
  "details": {
    "upstream": "artifacthub.io",
    "status": 503
  }
}
```

**Database Error** (500):
```json
{
  "success": false,
  "error": "Failed to create chart revision",
  "code": "DATABASE_ERROR"
}
```

### Tool-Side Error Handling

All AI SDK tools should handle errors consistently:

```typescript
// lib/ai/tools/utils.ts

export async function callGoEndpoint<T>(
  endpoint: string,
  body: Record<string, unknown>
): Promise<T> {
  const response = await fetch(`${process.env.GO_BACKEND_URL}${endpoint}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });

  const data = await response.json();

  if (!response.ok || !data.success) {
    const errorMessage = data.error || `Request failed: ${response.statusText}`;
    const errorCode = data.code || 'UNKNOWN_ERROR';
    throw new Error(`[${errorCode}] ${errorMessage}`);
  }

  return data as T;
}
```

**Usage in tools**:
```typescript
export const createChart = tool({
  // ...
  execute: async (params) => {
    return callGoEndpoint<CreateChartResponse>('/api/tools/charts/create', params);
  },
});
```

### New File

| File | Priority |
|------|----------|
| `pkg/api/errors.go` | MUST |
| `lib/ai/tools/utils.ts` | MUST |

---

## Updated Success Criteria

### Must Pass (Revised)

- [ ] "Create a nginx chart" works via AI SDK chat
- [ ] createChart tool invokes Go endpoint
- [ ] Chart created in database
- [ ] **latestSubchartVersion tool returns valid version data** ← NEW
- [ ] **latestKubernetesVersion tool returns valid version data** ← NEW
- [ ] No @anthropic-ai/sdk in node_modules
- [ ] Integration test passes
- [ ] **Error responses follow standard format** ← NEW

### Quality Gates (Revised)

- [ ] All **6 tools** implemented (was 4)
- [ ] System prompts guide tool usage
- [ ] ARCHITECTURE.md updated
- [ ] **Error handling utility shared across tools** ← NEW

---

## Summary of Changes

| Change | Impact |
|--------|--------|
| Add `latestSubchartVersion` tool | +1 TypeScript file, shared Go handler |
| Add `latestKubernetesVersion` tool | +1 TypeScript file, shared Go handler |
| Add `pkg/api/handlers/versions.go` | +1 Go file with 2 handlers |
| Add `pkg/api/errors.go` | +1 Go file for error utilities |
| Add `lib/ai/tools/utils.ts` | +1 TypeScript file for shared error handling |
| Update tool registration | 6 tools instead of 4 |
| Update system prompt | Document 2 additional tools |

**Net additions**: 5 new files, ~200 lines of code

---

*End of Changes Document*
