# PRD4: Chartsmith Validation Agent & Live Provider Switching
## Technical Specification Document

**Version**: 1.2  
**PR**: PR2 of 2  
**Timeline**: Days 4-6 (3 days)  
**Prerequisite**: PR1 merged

---

## Technical Overview

### Objective
Implement a Go-based validation pipeline exposed as an API endpoint, integrate with Vercel AI SDK tool calling, and enhance provider switching for live mid-conversation changes.

### Approach
- New Go validation package orchestrating helm and kube-score
- REST API endpoint for validation requests
- AI SDK tool definition calling Go backend
- Enhanced provider switcher component with state preservation

### Key Technical Decisions
- Validation runs in Go backend (not Node.js) to leverage Go's exec capabilities and align with learning objectives
- Sequential pipeline (not parallel) for simpler error handling and debugging
- kube-score failure is non-fatal to ensure partial results are always returned
- Provider switching preserves full message history client-side (no server state)

---

## System Architecture

```
┌─────────────────┐                    ┌─────────────────┐
│   React Chat    │                    │  Next.js API    │
│   + Tool UI     │ ◄──────────────────│   + Tools       │
└────────┬────────┘                    └────────┬────────┘
         │ Tool Call                            │ HTTP
         │                             ┌────────▼────────┐
         │                             │   Go Backend    │
         │                             │  /api/validate  │
         │                             └────────┬────────┘
         │                             ┌────────▼────────┐
         │                             │  Validation Pipeline
         │                             │  lint → template → kube-score
         └─────────────────────────────┴─────────────────┘
```

### Data Flow
1. User asks "validate my chart" in chat
2. AI SDK recognizes intent, invokes validateChart tool
3. Tool execute() sends POST to Go backend /api/validate
4. Go backend runs helm lint → helm template → kube-score
5. Results aggregated and returned as JSON
6. Tool result rendered in chat via ValidationResults component
7. AI interprets results and provides natural language explanation

---

## Go Backend Specification

### Package Structure

```
pkg/
├── api/
│   └── validate.go          # HTTP handler for /api/validate
├── validation/
│   ├── pipeline.go          # RunValidation orchestration
│   ├── helm.go              # runHelmLint, runHelmTemplate
│   ├── kubescore.go         # runKubeScore
│   ├── parser.go            # Output parsing utilities
│   └── types.go             # All type definitions
```

### Core Types

**ValidationRequest** - Input to the pipeline:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| ChartPath | string | Yes | Absolute or relative path to chart directory |
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

**Function**: `RunValidation(request ValidationRequest) (ValidationResult, error)`

The pipeline orchestrates three validation stages sequentially. Each stage can short-circuit the pipeline on critical failure.

**Stage 1 - Helm Lint**:
- Execute `helm lint <chartPath>` with optional `--values <tempfile>` if Values provided
- Add `--quiet` flag to suppress info messages (focus on warnings/errors)
- Parse stdout line-by-line, looking for `[ERROR]`, `[WARNING]`, `[INFO]` prefixes
- Extract file path and line number using regex (format: `templates/file.yaml:42`)
- If any [ERROR] found, set LintResult.Status = "fail"
- If critical errors exist AND StrictMode is false, continue to next stage
- If critical errors exist AND StrictMode is true, return early with overall "fail"

**Stage 2 - Helm Template**:
- Only runs if Stage 1 completed (even with warnings)
- Execute `helm template validation-check <chartPath>` with optional values and kube-version
- Capture stdout (rendered YAML) and stderr (errors)
- If exit code non-zero, parse stderr for error details, return "fail"
- On success, count YAML documents by counting `---` separators
- Store rendered YAML in memory for Stage 3
- Do NOT write rendered output to disk permanently (security)

**Stage 3 - Kube-Score**:
- Only runs if Stage 2 succeeded
- Write rendered YAML to temp file with .yaml extension
- Execute `kube-score score <tempfile> --output-format json`
- Delete temp file immediately after execution (defer)
- Parse JSON output - array of objects, each with Checks array
- For each check, map Grade to severity: 1=critical, 5=warning, 10=pass
- Calculate score: (passedChecks / totalChecks) * 10, rounded
- Generate suggestions based on check name (lookup table)
- If kube-score binary not found, log warning and skip (non-fatal)

**Overall Status Determination**:
- "fail" if any stage has Status "fail" OR any issue has Severity "critical"
- "warning" if any issue has Severity "warning" but no critical
- "pass" if all stages pass and no issues

---

### Helm Integration Details

**File**: `pkg/validation/helm.go`

**runHelmLint(chartPath string, values map[string]interface{}) (LintResult, error)**:

Build argument list starting with ["lint", chartPath]. If values map is non-empty, serialize to YAML, write to temp file, append ["--values", tempFilePath] to args, and defer deletion of temp file. Append "--quiet" to reduce noise.

Execute using exec.Command("helm", args...). Capture CombinedOutput() to get both stdout and stderr. Check exit code - helm lint returns non-zero on errors.

Parse output using parseHelmLintOutput helper. For each line, check prefix to determine severity. Extract message by stripping prefix. Use regex to find file:line patterns. Create ValidationIssue for each finding.

**runHelmTemplate(chartPath string, values map[string]interface{}, kubeVersion string) (TemplateResult, []byte, error)**:

Build argument list: ["template", "validation-check", chartPath]. The release name "validation-check" is arbitrary but required. Add values file if provided. Add ["--kube-version", kubeVersion] if specified.

Execute and capture stdout separately from stderr. On error, parse stderr to extract meaningful error message (often includes file and line). Return TemplateResult with Status "fail" and parsed issues.

On success, count documents in stdout by splitting on "\n---\n" pattern. Return TemplateResult with stats and the raw YAML bytes for kube-score.

---

### Kube-Score Integration Details

**File**: `pkg/validation/kubescore.go`

**runKubeScore(renderedYAML []byte, kubeVersion string) (ScoreResult, error)**:

First, check if kube-score is installed using exec.LookPath("kube-score"). If not found, return empty ScoreResult with Status "skipped" and log warning - do not fail the pipeline.

Write renderedYAML to a temp file. Use os.CreateTemp with pattern "chart-*.yaml". Defer os.Remove to ensure cleanup.

Build args: ["score", tempFilePath, "--output-format", "json"]. If kubeVersion provided, add ["--kubernetes-version", kubeVersion].

Execute and capture output. Parse JSON into []KubeScoreObject structure where each object has ObjectName and Checks array.

**Grade Mapping**:
| kube-score Grade | Our Severity |
|------------------|--------------|
| 1 (CRITICAL) | "critical" |
| 5 (WARNING) | "warning" |
| 7 (OK) | (skip - passed) |
| 10 (OK) | (skip - passed) |

**Suggestion Generation**: Maintain a map of check names to fix suggestions:
- "container-resources" → "Add resources.limits.memory and resources.limits.cpu"
- "container-security-context" → "Set securityContext.runAsNonRoot: true"
- "container-image-tag" → "Use specific image tag instead of :latest"
- "pod-probes" → "Add readinessProbe and livenessProbe to containers"
- "pod-networkpolicy" → "Create a NetworkPolicy for this namespace"
- (default) → "Review kube-score documentation for this check"

---

### API Handler

**File**: `pkg/api/validate.go`

**Handler Registration**: Register with router as POST /api/validate

**Request Handling**:
1. Check method is POST, return 405 if not
2. Set response Content-Type to application/json
3. Decode request body into ValidationRequest struct
4. If decode fails, return 400 with error details
5. Validate ChartPath is provided and non-empty
6. Sanitize ChartPath (see Security section)
7. Verify Chart.yaml exists at path, return 400 if not
8. Create context with 30-second timeout
9. Call validation.RunValidation(ctx, request)
10. If context deadline exceeded, return 504 Gateway Timeout
11. If other error, return 500 with error message
12. On success, return 200 with {"validation": result}

---

## Frontend Specification

### Validation Tool Definition

**File**: `src/lib/ai/tools/validateChart.ts`

**Tool Configuration**:
- **name**: "validateChart"
- **description**: "Validate a Helm chart for syntax errors, template rendering issues, and Kubernetes best practices. Use this tool when the user asks to validate, check, lint, verify, or review their chart, or when they ask about errors, issues, or problems with the chart."

**Input Schema** (Zod):
- chartPath: z.string().describe("Path to the Helm chart directory")
- values: z.record(z.any()).optional().describe("Values to override for template rendering")
- strictMode: z.boolean().optional().default(false).describe("Treat warnings as failures")
- kubeVersion: z.string().optional().describe("Target Kubernetes version")

**Execute Function**: Construct request body from input parameters. POST to GO_BACKEND_URL + "/api/validate" with JSON body. Handle non-ok responses by throwing error with message. Parse and return JSON response.

**Registration**: Export from tools/index.ts and include in chartsmithTools object passed to streamText().

### ValidationResults Component

**File**: `src/components/chat/ValidationResults.tsx`

**Props**: result (ValidationResult object from tool response)

**Internal State**:
- expandedIssues: Set<string> - tracks which issues are expanded
- showFullDetails: boolean - toggles full kube-score output

**Rendering Structure**:

Header section displays overall status with appropriate icon (✅ pass, ⚠️ warning, ❌ fail) and colored badge. Include timestamp and duration for context.

Summary line shows counts: "{N} critical, {M} warnings found" or "All checks passed" if clean.

Issues list renders all issues sorted by severity (critical first, then warning, then info). Each issue row shows:
- Severity icon matching the level
- Check name or message as primary text
- File:line location as secondary text (if available)
- Expandable section with full message and suggestion

Footer section shows kube-score summary if available: "Best Practices Score: {N}/10 ({passed}/{total} checks)". Include action buttons: "Show full details" expands raw output, "Validate again" triggers new validation.

**Interactions**:
- Click issue row to toggle expanded state
- Click "Show full details" to see complete tool output
- Click "Validate again" to re-invoke tool (requires parent callback)

### LiveProviderSwitcher Component

**File**: `src/components/chat/LiveProviderSwitcher.tsx`

**Props**:
- currentProvider: ProviderConfig
- onSwitch: (provider: ProviderConfig) => void
- providers: ProviderConfig[] (defaults to AVAILABLE_PROVIDERS)

**Internal State**:
- isOpen: boolean - dropdown visibility
- isTransitioning: boolean - brief indicator during switch

**Behavior**:

Render as compact button showing current provider icon/name with dropdown chevron. Position in chat header, right-aligned.

On click, toggle isOpen state. When open, render dropdown below button with all providers listed. Current provider shows checkmark. Clicking a different provider calls onSwitch, sets isTransitioning true for 300ms (visual feedback), then closes dropdown.

Click outside dropdown or pressing Escape closes it. Support keyboard navigation (arrow keys, Enter to select).

**Styling**: Match existing Chartsmith design system. Use subtle animation for dropdown open/close. Transition indicator could be brief spinner or color pulse.

### Chat Component Updates

**File**: Modify existing Chat component from PR1

**Changes from PR1**:

1. **Provider Selector Location**: In PR1, ProviderSelector only shows when messages empty. In PR2, replace with LiveProviderSwitcher that's always visible in chat header.

2. **Provider Switch Handler**: Remove the "disabled after first message" logic. Now handleProviderSwitch simply updates selectedProvider state - the next API call will use the new provider automatically since provider is passed in useChat body.

3. **Tool Result Rendering**: In message part rendering, add special case for validateChart tool results. When part.type === "tool-result" and part.toolName === "validateChart", render ValidationResults component with part.result.

4. **Validation Progress**: When part.type === "tool-call" and part.toolName === "validateChart", render a progress indicator showing "Validating chart..." with spinner.

### Tool Result Rendering Details

**In Message component**, extend the part rendering switch:

For "tool-call" parts: Check toolName. If "validateChart", render ValidationInProgress component showing animated indicator with text "Running validation..." and optional step progress if we track it. For other tools, use existing generic ToolCall renderer.

For "tool-result" parts: Check toolName. If "validateChart", render ValidationResults with the result data. Ensure result is parsed correctly - it may be nested under a "validation" key. For other tools, use existing generic ToolResult renderer.

---

## API Contract

### POST /api/validate

**Request Headers**:
- Content-Type: application/json
- Authorization: Bearer <token> (if auth enabled)

**Request Body**:
```json
{
  "chartPath": "/path/to/chart",
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
            "message": "Container does not have resource limits defined",
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
- 400: Invalid request (missing chartPath, invalid path, Chart.yaml not found)
- 405: Method not allowed (non-POST request)
- 500: Internal error (validation pipeline failure)
- 504: Timeout (validation exceeded 30 seconds)

---

## Testing Strategy

### Go Unit Tests

**Pipeline Tests** (`pipeline_test.go`):
| Test | Setup | Assertion |
|------|-------|-----------|
| Valid chart passes | Create temp chart with valid structure | OverallStatus == "pass" |
| Lint failure stops pipeline | Chart missing required Chart.yaml fields | OverallStatus == "fail", only HelmLint populated |
| Template failure captured | Chart with invalid template syntax | OverallStatus == "fail", TemplateResult has issues |
| Kube-score warnings aggregated | Valid chart without resource limits | OverallStatus == "warning", issues contain container-resources |
| Kube-score missing is non-fatal | Mock LookPath to return error | Pipeline completes, KubeScore nil or skipped |
| Timeout handled | Mock slow execution | Context deadline error returned |

**Parser Tests** (`parser_test.go`):
| Test | Input | Assertion |
|------|-------|-----------|
| Lint ERROR extracted | "[ERROR] Chart.yaml: version required" | Issue with severity critical |
| Lint WARNING extracted | "[WARNING] templates/: directory empty" | Issue with severity warning |
| File:line parsed | "templates/deployment.yaml:42: error" | File and Line fields populated |
| Kube-score JSON parsed | Sample JSON with mixed grades | Correct issue count and severities |

**Handler Tests** (`validate_test.go`):
| Test | Request | Expected Response |
|------|---------|-------------------|
| Valid request | POST with valid chartPath | 200 with validation result |
| Missing chartPath | POST with empty body | 400 with error message |
| Invalid path | POST with "../../../etc" | 400 path validation error |
| GET request | GET /api/validate | 405 method not allowed |

### Frontend Tests

**Tool Definition Tests**:
- Schema accepts valid input with all fields
- Schema accepts minimal input (chartPath only)
- Schema rejects missing chartPath
- Execute function calls correct endpoint
- Execute function handles error responses

**ValidationResults Tests**:
- Renders pass state with green indicator
- Renders warning state with yellow indicator
- Renders fail state with red indicator
- Displays correct issue count
- Issues sorted by severity
- Click expands issue details
- Show details button reveals full output

**LiveProviderSwitcher Tests**:
- Shows current provider name
- Dropdown opens on click
- Current provider has checkmark
- Selecting new provider calls onSwitch
- Dropdown closes after selection
- Escape key closes dropdown

---

## Error Handling

### Go Backend Errors

| Scenario | Detection | Response |
|----------|-----------|----------|
| helm not installed | exec.LookPath returns error | 500 with "helm CLI not found" |
| kube-score not installed | exec.LookPath returns error | Continue without kube-score, log warning |
| Chart path doesn't exist | os.Stat returns error | 400 with "chart path does not exist" |
| Chart.yaml missing | os.Stat on Chart.yaml fails | 400 with "not a valid Helm chart" |
| Validation timeout | context.DeadlineExceeded | 504 with "validation timed out" |
| helm lint fails | Non-zero exit code | Include in results, may continue |
| helm template fails | Non-zero exit code | Return fail status with parsed error |
| JSON parse error | json.Unmarshal fails | 500 with parse error details |

### Frontend Error Handling

| Scenario | Detection | User Experience |
|----------|-----------|-----------------|
| Network error | fetch throws | "Validation service unavailable" message |
| 4xx response | response.ok false | Display error message from response |
| 5xx response | response.status >= 500 | "Validation failed, please try again" |
| Malformed response | JSON parse fails | Generic error, log details to console |
| Tool timeout | No response in 35s | "Validation is taking longer than expected" |

---

## Security Considerations

### Path Traversal Prevention

Before using chartPath in any exec.Command:
1. Apply filepath.Clean() to normalize path
2. Check for ".." components - reject if present
3. Resolve to absolute path using filepath.Abs()
4. Verify path is within allowed base directory
5. Confirm Chart.yaml exists at resolved path

### Command Injection Prevention

Never construct shell commands with string concatenation. Always use exec.Command with separate argument array. The chartPath is passed as a single argument, not interpolated into a string.

Correct: `exec.Command("helm", "lint", chartPath)`
Wrong: `exec.Command("sh", "-c", "helm lint " + chartPath)`

### Temp File Security

When writing values or rendered YAML to temp files:
- Use os.CreateTemp with restrictive permissions (0600)
- Generate random filename (os.CreateTemp handles this)
- Delete immediately after use with defer
- Never use user-provided filenames

### Resource Limits

- Pipeline timeout: 30 seconds maximum
- Output size: Truncate if > 1MB
- Concurrent validations: Consider limiting per-user (future)

---

## Performance Considerations

### Expected Timing

| Stage | Typical Duration | Maximum |
|-------|------------------|---------|
| helm lint | 100-500ms | 5s |
| helm template | 200-1000ms | 10s |
| kube-score | 500-2000ms | 15s |
| Total pipeline | 1-3s | 30s |

### Optimization Opportunities (Future)

- Cache rendered YAML for repeated validation requests
- Parallelize kube-score with other post-template checks
- Stream results as each stage completes
- Skip unchanged validation (hash chart contents)

### Frontend Performance

- Throttle re-renders during streaming with experimental_throttle
- Lazy-load ValidationResults component
- Virtualize issue list if > 50 issues

---

## Migration Sequence

### Day 4: Go Backend

**Morning (4 hours)**:
1. Create pkg/validation/ directory structure
2. Define all types in types.go
3. Implement runHelmLint in helm.go
4. Implement runHelmTemplate in helm.go
5. Write unit tests for helm functions
6. Test manually with sample charts

**Afternoon (4 hours)**:
1. Implement runKubeScore in kubescore.go
2. Implement output parsers in parser.go
3. Implement RunValidation pipeline in pipeline.go
4. Write integration tests for full pipeline
5. Test with charts that have known issues

### Day 5: API & Frontend

**Morning (4 hours)**:
1. Implement /api/validate handler in validate.go
2. Register route in existing router
3. Test endpoint with curl/Postman
4. Create validateChart tool definition
5. Register tool in streamText configuration

**Afternoon (4 hours)**:
1. Build ValidationResults component
2. Build LiveProviderSwitcher component
3. Update Chat component to use LiveProviderSwitcher
4. Add tool result rendering for validateChart
5. Test tool invocation end-to-end

### Day 6: Integration & Polish

**Morning (4 hours)**:
1. End-to-end testing of complete validation flow
2. Test provider switching during validation
3. Test error scenarios (missing tools, bad charts)
4. Fix bugs discovered during testing

**Afternoon (4 hours)**:
1. Write/update frontend tests
2. Update ARCHITECTURE.md documentation
3. Code cleanup and comments
4. Final review and PR preparation

---

## Validation Checklist

### Go Backend
- [ ] pkg/validation/ package structure created
- [ ] All types defined in types.go
- [ ] runHelmLint implemented and tested
- [ ] runHelmTemplate implemented and tested
- [ ] runKubeScore implemented and tested
- [ ] RunValidation pipeline orchestrates all stages
- [ ] /api/validate endpoint registered and working
- [ ] Path validation prevents traversal attacks
- [ ] Timeout handling works correctly
- [ ] Missing kube-score handled gracefully

### Frontend
- [ ] validateChart tool definition complete
- [ ] Tool registered in streamText configuration
- [ ] ValidationResults component renders all states
- [ ] LiveProviderSwitcher replaces PR1 selector
- [ ] Provider switch preserves conversation history
- [ ] Tool call shows progress indicator
- [ ] Tool result renders ValidationResults
- [ ] Error states display user-friendly messages

### Integration
- [ ] "Validate my chart" triggers tool correctly
- [ ] Results display in chat with proper formatting
- [ ] AI provides natural language interpretation
- [ ] All PR1 functionality still works
- [ ] Tests pass for both Go and frontend

---

## Appendix: Tool Output Reference

### helm lint Output Format
```
==> Linting ./mychart
[INFO] Chart.yaml: icon is recommended
[WARNING] templates/deployment.yaml: object name does not conform
[ERROR] templates/service.yaml: unable to parse YAML
Error: 1 chart(s) linted, 1 chart(s) failed
```

### kube-score JSON Structure
```json
[{
  "object_name": "apps/v1/Deployment/my-app",
  "file_name": "rendered.yaml",
  "checks": [{
    "check": {
      "name": "container-resources",
      "id": "container-resources",
      "comment": "Makes sure containers have resource limits set"
    },
    "grade": 1,
    "skipped": false,
    "comments": [{"summary": "Container has no resource limits"}]
  }]
}]
```

### Common Validation Issues

| Check | Severity | Message | Suggestion |
|-------|----------|---------|------------|
| container-resources | Critical | No resource limits | Add resources.limits |
| container-security-context | Critical | Runs as root | Set runAsNonRoot: true |
| container-image-tag | Warning | Using :latest | Pin specific version |
| pod-probes | Warning | No readiness probe | Add readinessProbe |
| pod-networkpolicy | Warning | No NetworkPolicy | Create NetworkPolicy |
| deployment-replicas | Warning | Only 1 replica | Increase for HA |

---

*Document End - PRD4: PR2 Technical Specification*
