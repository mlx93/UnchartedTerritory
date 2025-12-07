# PR4: Chartsmith Validation Agent & Live Provider Switching
## Technical Specification Document

**Version**: 1.0
**PR**: PR4 (formerly PR2)
**Status**: Ready for Implementation
**Prerequisite**: PR3.0/3.1 merged

---

## Technical Overview

### Objective
Implement a Go-based validation pipeline, integrate with existing AI SDK tool infrastructure, and complete provider switching for live mid-conversation changes.

### Approach
- New Go validation package orchestrating helm and kube-score
- New tool following established factory pattern in `lib/ai/tools/`
- Provider state management in `useAISDKChatAdapter`
- LiveProviderSwitcher component integrated into ChatContainer

### Key Technical Decisions
- Validation runs in Go backend using established HTTP endpoint pattern
- Sequential pipeline (not parallel) for simpler error handling
- kube-score failure is non-fatal to ensure partial results
- Provider switching preserves full message history client-side
- validateChart is a **non-buffered** tool (read-only, executes immediately)

---

## System Architecture

### Current State (After PR3.1)

```
+-----------------------------------------------------------------------------+
|                        AFTER PR3.1 (Before PR4)                              |
+-----------------------------------------------------------------------------+
|                                                                              |
|  +-------------------+     useChat Hook    +-------------------+             |
|  |   Chat UI         | ------------------> |  /api/chat        | --> OpenRouter
|  |  (AI SDK-based)   | <------------------ |   (streamText)    |             |
|  |  @ai-sdk/react    |   Data Stream       |   + 5 AI SDK      |             |
|  +-------------------+                     |     tools         |             |
|                                            +---------+---------+             |
|                                                      | HTTP (tool calls)     |
|  - - - - - - - - - - - - - - - - - - - - - - - - - - | - - - - - - - - - -   |
|  GO BACKEND (HTTP SERVER on :8080 + PostgreSQL queue)                        |
|  - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -   |
|                                                      |                       |
|  +-------------------+              +----------------v-----------------+     |
|  | Workspace APIs    |  pg_notify   |  Go HTTP Server (:8080)          |     |
|  | (homepage flow)   | -----------> |  POST /api/tools/editor          |     |
|  +-------------------+              |  POST /api/tools/context         |     |
|                                     |  POST /api/tools/versions/*      |     |
|                                     |  POST /api/intent/classify       |     |
|                                     |  POST /api/plan/*                |     |
|                                     |  POST /api/conversion/start      |     |
|                                     +----------------------------------+     |
|                                                                              |
+-----------------------------------------------------------------------------+
```

### PR4 Additions

```
+-------------------+                    +-------------------+
|   Chat UI         |     useChat        |  /api/chat        |
|   (AI SDK)        | -----------------> |  (streamText)     |
|   + Provider      |                    |  + validateChart  | <-- NEW tool
|     Switcher      |                    |    tool           |
+-------------------+                    +---------+---------+
                                                   | Tool calls validateChart
                                         +---------v---------+
                                         |  validateChart    |
                                         |  tool execute()   |
                                         +---------+---------+
                                                   | HTTP POST (callGoEndpoint)
                                         +---------v---------+
                                         |  Go Backend       |
                                         |  POST /api/validate| <-- NEW endpoint
                                         +---------+---------+
                                         +---------v---------+
                                         |  Validation       |
                                         |  Pipeline         |
                                         |  lint -> template |
                                         |  -> kube-score    |
                                         +-----------------+
```

### Data Flow
1. User asks "validate my chart" in chat
2. AI SDK recognizes intent, invokes validateChart tool
3. Tool execute() sends POST to Go backend `/api/validate` via `callGoEndpoint()`
4. Go backend runs helm lint -> helm template -> kube-score
5. Results aggregated and returned as JSON
6. Tool result rendered in chat via ValidationResults component
7. AI interprets results and provides natural language explanation

---

## Go Backend Specification

### Existing Structure (Reference)

```
pkg/
├── api/
│   ├── server.go              # HTTP server setup, route registration
│   └── handlers/
│       ├── context.go         # POST /api/tools/context
│       ├── editor.go          # POST /api/tools/editor
│       ├── versions.go        # POST /api/tools/versions/*
│       ├── intent.go          # POST /api/intent/classify
│       ├── plan.go            # POST /api/plan/*
│       ├── conversion.go      # POST /api/conversion/start
│       ├── render.go          # POST /api/render/trigger
│       └── response.go        # Error response utilities
├── workspace/                 # Workspace/chart/file operations
└── realtime/                  # Centrifugo integration
```

### NEW Package Structure for PR4

```
pkg/
├── api/
│   ├── server.go              # MODIFIED - Add /api/validate route
│   └── handlers/
│       └── validate.go        # NEW - HTTP handler
├── validation/                # NEW - Validation logic
│   ├── pipeline.go            # RunValidation orchestration
│   ├── helm.go                # runHelmLint, runHelmTemplate
│   ├── kubescore.go           # runKubeScore
│   ├── parser.go              # Output parsing utilities
│   └── types.go               # All type definitions
```

### Core Types

**ValidationRequest** - Input to the pipeline:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| WorkspaceID | string | Yes | Workspace to validate |
| RevisionNumber | int | Yes | Revision to validate |
| Values | map[string]interface{} | No | Values to override during template rendering |
| StrictMode | bool | No | If true, treat warnings as failures |
| KubeVersion | string | No | Target K8s version (e.g., "1.28") |

**ValidationResult** - Pipeline output:
| Field | Type | Description |
|-------|------|-------------|
| OverallStatus | string | "pass", "warning", or "fail" |
| Timestamp | time.Time | When validation started |
| DurationMs | int64 | Total pipeline duration |
| Results | ValidationResults | Nested results from each stage |

**ValidationResults** - Container for stage results:
| Field | Type | Description |
|-------|------|-------------|
| HelmLint | LintResult | Results from helm lint stage |
| HelmTemplate | TemplateResult | Results from helm template stage |
| KubeScore | ScoreResult | Results from kube-score stage (may be nil) |

**ValidationIssue** - Individual finding:
| Field | Type | Description |
|-------|------|-------------|
| Severity | string | "critical", "warning", or "info" |
| Source | string | "helm_lint", "helm_template", or "kube_score" |
| Resource | string | K8s resource name (kube-score only) |
| Check | string | Check/rule name that triggered issue |
| Message | string | Human-readable description |
| File | string | Source file path (when available) |
| Line | int | Line number (when available, 0 if unknown) |
| Suggestion | string | Recommended fix action |

**LintResult**:
| Field | Type | Description |
|-------|------|-------------|
| Status | string | "pass" or "fail" |
| Issues | []ValidationIssue | All lint findings |

**TemplateResult**:
| Field | Type | Description |
|-------|------|-------------|
| Status | string | "pass" or "fail" |
| RenderedResources | int | Count of K8s resources generated |
| OutputSizeBytes | int | Size of rendered YAML |
| Issues | []ValidationIssue | Template errors if any |

**ScoreResult**:
| Field | Type | Description |
|-------|------|-------------|
| Status | string | "pass", "warning", or "fail" |
| Score | int | 0-10 score (passed/total * 10) |
| TotalChecks | int | Total checks executed |
| PassedChecks | int | Checks that passed |
| Issues | []ValidationIssue | Failed checks with details |

---

### Validation Pipeline Logic

**File**: `pkg/validation/pipeline.go`

**Function**: `RunValidation(ctx context.Context, request ValidationRequest) (ValidationResult, error)`

The pipeline orchestrates three validation stages sequentially.

**Stage 1 - Helm Lint**:
- Get chart files from workspace using `workspace.ListCharts()`
- Write chart to temp directory structure
- Execute `helm lint <chartPath>` with optional values
- Parse stdout for `[ERROR]`, `[WARNING]`, `[INFO]` prefixes
- If critical errors AND StrictMode, return early

**Stage 2 - Helm Template**:
- Only runs if Stage 1 completed
- Execute `helm template validation-check <chartPath>`
- Capture stdout (rendered YAML) and stderr (errors)
- On success, count YAML documents
- Store rendered YAML in memory for Stage 3

**Stage 3 - Kube-Score**:
- Only runs if Stage 2 succeeded
- Write rendered YAML to temp file
- Execute `kube-score score <tempfile> --output-format json`
- Delete temp file after execution
- Parse JSON, map Grade to severity
- If kube-score not found, skip (non-fatal)

**Overall Status Determination**:
- "fail" if any stage fails OR any issue has Severity "critical"
- "warning" if any issue has Severity "warning" but no critical
- "pass" if all stages pass and no issues

---

### Helm Integration Details

**File**: `pkg/validation/helm.go`

**runHelmLint(chartPath string, values map[string]interface{}) (LintResult, error)**:

Build argument list: `["lint", chartPath, "--quiet"]`. If values provided, serialize to YAML temp file, add `--values` flag. Execute with `exec.Command`. Parse output using regex for severity prefixes and file:line patterns.

**runHelmTemplate(chartPath string, values map[string]interface{}, kubeVersion string) (TemplateResult, []byte, error)**:

Build argument list: `["template", "validation-check", chartPath]`. Add values and kube-version if provided. Execute and capture stdout separately. On success, count documents by `---` separators. Return raw YAML bytes for kube-score.

---

### Kube-Score Integration Details

**File**: `pkg/validation/kubescore.go`

**runKubeScore(renderedYAML []byte, kubeVersion string) (ScoreResult, error)**:

Check if kube-score installed via `exec.LookPath()`. If not found, return empty ScoreResult with Status "skipped".

Write renderedYAML to temp file. Execute `kube-score score <tempfile> --output-format json`. Parse JSON output.

**Grade Mapping**:
| kube-score Grade | Our Severity |
|------------------|--------------|
| 1 (CRITICAL) | "critical" |
| 5 (WARNING) | "warning" |
| 7-10 (OK) | (skip - passed) |

**Suggestion Generation**: Map check names to fix suggestions:
- "container-resources" -> "Add resources.limits.memory and resources.limits.cpu"
- "container-security-context" -> "Set securityContext.runAsNonRoot: true"
- "container-image-tag" -> "Use specific image tag instead of :latest"
- "pod-probes" -> "Add readinessProbe and livenessProbe"
- (default) -> "Review kube-score documentation"

---

### API Handler

**File**: `pkg/api/handlers/validate.go`

**Handler Registration** (in `server.go`):
```go
// Add after existing routes (~line 44)
mux.HandleFunc("POST /api/validate", handlers.ValidateChart)
```

**Request Handling**:
1. Decode request body into ValidationRequest
2. Validate WorkspaceID and RevisionNumber provided
3. Create context with 30-second timeout
4. Get chart files from workspace
5. Write to temp directory
6. Call `validation.RunValidation(ctx, request)`
7. Return JSON response using `writeJSON()`

**Security**:
- Use existing workspace access patterns from `handlers/editor.go`
- No path traversal risk since we use workspace ID, not raw paths

---

## Frontend Specification

### Existing Infrastructure (Reference)

**Tool Factories** (`chartsmith-app/lib/ai/tools/`):
- `index.ts` - `createTools()` and `createBufferedTools()`
- `textEditor.ts` - Example factory pattern
- `utils.ts` - `callGoEndpoint()` helper

**Chat Adapter** (`chartsmith-app/hooks/useAISDKChatAdapter.ts`):
- Lines 138-148: `getChatBody()` - **Currently hardcodes provider/model**
- Lines 273-276: `sendMessage()` with body

**Chat Container** (`chartsmith-app/components/ChatContainer.tsx`):
- Lines 214-276: Role selector dropdown (persona)
- No provider selector currently

### Validation Tool Definition

**File**: `chartsmith-app/lib/ai/tools/validateChart.ts` (NEW)

**Factory Function**:
```typescript
export function createValidateChartTool(
  authHeader: string | undefined,
  workspaceId: string,
  revisionNumber: number
) {
  return tool({
    description: "Validate a Helm chart for syntax errors, rendering issues, " +
      "and Kubernetes best practices. Use this when the user asks to validate, " +
      "check, lint, or verify their chart.",
    inputSchema: z.object({
      values: z.record(z.any()).optional()
        .describe("Values to override for template rendering"),
      strictMode: z.boolean().optional()
        .describe("Treat warnings as failures"),
      kubeVersion: z.string().optional()
        .describe("Target Kubernetes version")
    }),
    execute: async (params) => {
      return await callGoEndpoint<ValidationResponse>(
        '/api/validate',
        {
          workspaceId,
          revisionNumber,
          values: params.values,
          strictMode: params.strictMode,
          kubeVersion: params.kubeVersion,
        },
        authHeader
      );
    }
  });
}
```

**Registration** (in `tools/index.ts`):
```typescript
import { createValidateChartTool } from './validateChart';

export function createTools(...) {
  return {
    ...existingTools,
    validateChart: createValidateChartTool(authHeader, workspaceId, revisionNumber),
  };
}

// Also add to createBufferedTools() - but as non-buffered (read-only)
```

### ValidationResults Component

**File**: `chartsmith-app/components/chat/ValidationResults.tsx` (NEW)

**Props**:
- `result`: ValidationResult object from tool response

**Structure**:
Follow the pattern from `PlanChatMessage.tsx`:
- Header with overall status badge
- Summary stats (issue counts)
- Issues list sorted by severity
- Expandable details for each issue
- kube-score summary if available
- Action buttons

### LiveProviderSwitcher Component

**File**: `chartsmith-app/components/chat/LiveProviderSwitcher.tsx` (NEW)

**Option**: Could also extend existing `ProviderSelector.tsx` instead of creating new.

**Props**:
- `currentProvider`: Provider - currently selected
- `currentModel`: string - currently selected model
- `onSwitch`: (provider: Provider, model: string) => void

**Behavior**:
- Dropdown showing current model
- List of available providers/models
- Checkmark on current selection
- Click to switch

### useAISDKChatAdapter Changes

**File**: `chartsmith-app/hooks/useAISDKChatAdapter.ts`

**Current Problem** (lines 138-148):
```typescript
const getChatBody = useCallback(
  (messageId?: string) => ({
    provider: DEFAULT_PROVIDER,  // Hardcoded!
    model: DEFAULT_MODEL,        // Hardcoded!
    workspaceId,
    revisionNumber,
    persona: currentPersonaRef.current,
    chatMessageId: messageId,
  }),
  [workspaceId, revisionNumber]
);
```

**Required Changes**:

1. Add state for selected provider/model:
```typescript
const [selectedProvider, setSelectedProvider] = useState<string>(DEFAULT_PROVIDER);
const [selectedModel, setSelectedModel] = useState<string>(DEFAULT_MODEL);
```

2. Update getChatBody to use state:
```typescript
const getChatBody = useCallback(
  (messageId?: string) => ({
    provider: selectedProvider,
    model: selectedModel,
    // ...rest unchanged
  }),
  [workspaceId, revisionNumber, selectedProvider, selectedModel]
);
```

3. Expose switch function:
```typescript
const switchProvider = useCallback((provider: string, model: string) => {
  setSelectedProvider(provider);
  setSelectedModel(model);
}, []);

// Return from hook:
return {
  ...existingReturns,
  selectedProvider,
  selectedModel,
  switchProvider,
};
```

### ChatContainer Changes

**File**: `chartsmith-app/components/ChatContainer.tsx`

**Add LiveProviderSwitcher**:
- Import component
- Get `selectedProvider`, `selectedModel`, `switchProvider` from adapter
- Add LiveProviderSwitcher next to role selector in header

**Example**:
```typescript
<div className="flex items-center gap-2">
  <LiveProviderSwitcher
    currentProvider={selectedProvider}
    currentModel={selectedModel}
    onSwitch={switchProvider}
  />
  <RoleSelector ... /> // Existing persona selector
</div>
```

### Tool Result Rendering

**File**: `chartsmith-app/lib/chat/messageMapper.ts`

Add detection for validateChart tool results similar to existing patterns.

**File**: `chartsmith-app/components/ChatMessage.tsx` or `NewChartChatMessage.tsx`

Add conditional rendering for validation results:
```typescript
{hasValidationResult && (
  <ValidationResults result={validationResult} />
)}
```

---

## API Contract

### POST /api/validate

**Request**:
```json
{
  "workspaceId": "ws_123",
  "revisionNumber": 5,
  "values": { "replicaCount": 3 },
  "strictMode": false,
  "kubeVersion": "1.28"
}
```

**Success Response** (200):
```json
{
  "validation": {
    "overall_status": "warning",
    "timestamp": "2024-01-15T10:30:00Z",
    "duration_ms": 2500,
    "results": {
      "helm_lint": { "status": "pass", "issues": [] },
      "helm_template": {
        "status": "pass",
        "rendered_resources": 5,
        "output_size_bytes": 4096
      },
      "kube_score": {
        "status": "warning",
        "score": 6,
        "total_checks": 15,
        "passed_checks": 12,
        "issues": [
          {
            "severity": "critical",
            "source": "kube_score",
            "resource": "apps/v1/Deployment/my-app",
            "check": "container-resources",
            "message": "Container does not have resource limits",
            "file": "templates/deployment.yaml",
            "line": 24,
            "suggestion": "Add resources.limits.memory and resources.limits.cpu"
          }
        ]
      }
    }
  }
}
```

**Error Responses**:
- 400: Missing workspaceId/revisionNumber, workspace not found
- 500: Internal error (pipeline failure)
- 504: Timeout (exceeded 30 seconds)

---

## Testing Strategy

### Go Unit Tests

**Pipeline Tests** (`pipeline_test.go`):
| Test | Assertion |
|------|-----------|
| Valid chart passes | OverallStatus == "pass" |
| Lint failure stops pipeline | Only HelmLint populated |
| Template failure captured | TemplateResult has issues |
| Kube-score warnings aggregated | Issues contain container-resources |
| Kube-score missing is non-fatal | Pipeline completes |

**Handler Tests** (`validate_test.go`):
| Test | Expected Response |
|------|-------------------|
| Valid request | 200 with validation result |
| Missing workspaceId | 400 with error |
| Workspace not found | 400 with error |

### Frontend Tests

**Tool Tests**:
- Schema accepts valid input
- Execute calls correct endpoint
- Error handling works

**ValidationResults Tests**:
- Renders pass/warning/fail states correctly
- Issues sorted by severity
- Expand/collapse works

**LiveProviderSwitcher Tests**:
- Shows current provider
- Switch calls callback
- Dropdown opens/closes

---

## Error Handling

### Go Backend

| Scenario | Response |
|----------|----------|
| helm not installed | 500 "helm CLI not found" |
| kube-score not installed | Continue without, log warning |
| Workspace not found | 400 "workspace not found" |
| Validation timeout | 504 "validation timed out" |

### Frontend

| Scenario | User Experience |
|----------|-----------------|
| Network error | "Validation service unavailable" |
| 4xx response | Display error message |
| 5xx response | "Validation failed, please try again" |

---

## Security Considerations

### Workspace-Based Access
- Use workspaceId instead of raw file paths
- Leverage existing workspace access patterns from `handlers/editor.go`
- No path traversal risk

### Command Execution
- Use `exec.Command` with argument array (no shell)
- Never concatenate user input into commands

### Temp Files
- Use `os.CreateTemp` with restrictive permissions
- Delete immediately after use with defer

---

## Implementation Sequence

### Phase 1: Go Validation Package

**Files to Create**:
- `pkg/validation/types.go` - All type definitions
- `pkg/validation/helm.go` - Helm lint/template functions
- `pkg/validation/kubescore.go` - Kube-score integration
- `pkg/validation/parser.go` - Output parsing
- `pkg/validation/pipeline.go` - RunValidation orchestration

**Testing**: Unit tests for each function

### Phase 2: Go API Handler

**Files to Create**:
- `pkg/api/handlers/validate.go` - HTTP handler

**Files to Modify**:
- `pkg/api/server.go` - Add route registration

**Testing**: Integration tests with curl

### Phase 3: Frontend Tool

**Files to Create**:
- `chartsmith-app/lib/ai/tools/validateChart.ts` - Tool factory

**Files to Modify**:
- `chartsmith-app/lib/ai/tools/index.ts` - Register tool

**Testing**: Tool invocation works end-to-end

### Phase 4: Frontend Components

**Files to Create**:
- `chartsmith-app/components/chat/ValidationResults.tsx`
- `chartsmith-app/components/chat/LiveProviderSwitcher.tsx`

**Files to Modify**:
- `chartsmith-app/hooks/useAISDKChatAdapter.ts` - Add provider state
- `chartsmith-app/components/ChatContainer.tsx` - Add switcher

**Testing**: UI renders correctly, switching works

### Phase 5: Integration & Polish

- End-to-end testing
- Error scenarios
- Documentation updates

---

## Validation Checklist

### Go Backend
- [ ] `pkg/validation/` package created
- [ ] All types defined
- [ ] runHelmLint implemented
- [ ] runHelmTemplate implemented
- [ ] runKubeScore implemented
- [ ] RunValidation pipeline works
- [ ] `/api/validate` endpoint registered
- [ ] Timeout handling works
- [ ] Missing kube-score handled gracefully

### Frontend
- [ ] validateChart tool factory created
- [ ] Tool registered in createTools()
- [ ] ValidationResults component renders all states
- [ ] LiveProviderSwitcher works
- [ ] useAISDKChatAdapter has provider state
- [ ] ChatContainer has provider switcher

### Integration
- [ ] "Validate my chart" triggers tool
- [ ] Results display correctly
- [ ] AI provides interpretation
- [ ] Provider switching preserves history
- [ ] All existing functionality works

---

*Document End - PR4: Technical Specification*
