# PR2 Implementation Agent

## Your Mission
Implement PR2: Chart Validation Agent & Live Provider Switching for Chartsmith.

This is the **validation pipeline showcase** for the Uncharted Territory challenge—substantial new Go code that demonstrates mastery of the language through a real production feature.

---

## Primary Documents (READ FIRST - IN ORDER)
1. `PRDs/PR2_Tech_PRD.md` - Technical specification
2. `PRDs/PR2_Product_PRD.md` - Functional requirements
3. `PRDs/PR2_SUPPLEMENTAL_DETAILS.md` - Component pseudocode and details
4. `PRDs/CHARTSMITH_ARCHITECTURE_DECISIONS.md` - Architectural context

**Note**: `PRDs/` contains planning docs at workspace root. `chartsmith/` is the forked repo. All TypeScript in `chartsmith/chartsmith-app/`, all Go in `chartsmith/pkg/`.

## Codebase Reference
- `chartsmith/CLAUDE.md` - Codebase entry points and Go patterns
- `docs/PR1.5_COMPLETION_REPORT.md` - What PR1.5 built (patterns to follow)
- `memory-bank/systemPatterns.md` - Existing Go patterns

---

## PR1.5 Context (COMPLETE)

PR1.5 established these patterns you will extend:

### What Already Exists
| File | What It Does |
|------|--------------|
| `pkg/api/server.go` | HTTP server on port 8080 - **ADD YOUR ROUTE HERE** |
| `pkg/api/handlers/response.go` | `WriteJSON()`, `WriteError()` helpers - **REUSE THESE** |
| `lib/ai/tools/utils.ts` | `callGoEndpoint()` helper - **REUSE THIS** |
| `lib/ai/tools/index.ts` | `createTools()` factory - **ADD YOUR TOOL HERE** |
| `app/api/chat/route.ts` | Tools registered in `streamText` - **ADD YOUR TOOL HERE** |

### Established Patterns
- Go handlers return `{ success: true/false, ... }` JSON
- TypeScript tools use `callGoEndpoint()` for HTTP calls
- Tools created via factory functions with auth header closure
- Error codes: `VALIDATION_ERROR`, `NOT_FOUND`, `INTERNAL_ERROR`
- **AI SDK v5 uses `inputSchema` (NOT `parameters`)** for tool definitions

### Current Go Endpoints (4 total)
```
POST /api/tools/context    → handlers.GetChartContext
POST /api/tools/editor     → handlers.TextEditor
POST /api/tools/versions/subchart   → handlers.GetSubchartVersion
POST /api/tools/versions/kubernetes → handlers.GetKubernetesVersion
```

### Current Tool Count
4 tools working: getChartContext, textEditor, latestSubchartVersion, latestKubernetesVersion

**Note**: All 4 tools call Go HTTP endpoints (including getChartContext, which was changed from TypeScript-only due to Next.js bundling issues with the `pg` module).

---

## What You Are Building

### The Validation Pipeline (Go) - YOUR GO SHOWCASE

This is the substantial new Go code that demonstrates learning. Create:

```
pkg/validation/
├── pipeline.go      # RunValidation orchestration
├── helm.go          # runHelmLint, runHelmTemplate
├── kubescore.go     # runKubeScore
├── parser.go        # Output parsing
└── types.go         # All type definitions
```

### Pipeline Flow
1. `helm lint <chartPath>` → Parse output → Continue if pass
2. `helm template <chartPath>` → Parse output → Continue if pass  
3. `kube-score score <rendered.yaml> --output-format json` → Parse → Include results

**CRITICAL**: kube-score failure is NON-FATAL. Always return partial results even if one stage fails.

### Go Files to Create (6 new)

| File | Purpose |
|------|---------|
| `pkg/api/handlers/validate.go` | HTTP handler for `/api/validate` |
| `pkg/validation/pipeline.go` | Orchestrates lint → template → kube-score |
| `pkg/validation/helm.go` | `exec.Command` for helm CLI |
| `pkg/validation/kubescore.go` | `exec.Command` for kube-score CLI |
| `pkg/validation/parser.go` | Parse CLI outputs to structured types |
| `pkg/validation/types.go` | ValidationResult, ValidationIssue, etc. |

### TypeScript Files to Create (2 new)

| File | Purpose |
|------|---------|
| `lib/ai/tools/validateChart.ts` | AI SDK tool definition (use `inputSchema`, not `parameters`) |
| `components/chat/ValidationResults.tsx` | Render validation output in chat |

### TypeScript Files to Modify

| File | Changes |
|------|---------|
| `lib/ai/tools/index.ts` | Add validateChart to `createTools()` |
| `app/api/chat/route.ts` | validateChart now in tools object |
| `components/chat/AIMessageList.tsx` | Render ValidationResults for tool results |

---

## Security Requirements (CRITICAL)

### Path Traversal Prevention
Every handler that accepts a path MUST validate it:

```
1. filepath.Clean(path) - Normalize
2. Reject if strings.Contains(path, "..") 
3. filepath.Abs() - Get absolute path
4. Verify within allowed chart directory
5. Verify Chart.yaml exists
```

### Command Injection Prevention
NEVER construct shell commands with string concatenation:

```go
// CORRECT - Arguments as separate strings
exec.Command("helm", "lint", chartPath)

// WRONG - NEVER DO THIS
exec.Command("sh", "-c", "helm lint " + chartPath)
```

---

## Validation Pipeline Implementation

### `pkg/validation/types.go`

Define these types:
- `ValidationResult` - Overall result with all stages
- `ValidationStage` - Individual stage (lint, template, kubescore)
- `ValidationIssue` - Single issue with severity, message, location
- `Severity` - enum: error, warning, info

### `pkg/validation/pipeline.go`

Orchestration function:
```
func RunValidation(ctx context.Context, chartPath string) (*ValidationResult, error)
```

Stages run sequentially. If helm lint fails, skip template. If template fails, skip kube-score. Always return what you have.

### `pkg/validation/helm.go`

Two functions:
```
func runHelmLint(chartPath string) (*ValidationStage, error)
func runHelmTemplate(chartPath string) (renderedYAML string, *ValidationStage, error)
```

Parse helm output for warnings/errors. Return rendered YAML from template for kube-score input.

### `pkg/validation/kubescore.go`

```
func runKubeScore(renderedYAML string) (*ValidationStage, error)
```

Write rendered YAML to temp file, run `kube-score score <file> --output-format json`, parse JSON output.

---

## Live Provider Switching

### Key Change from PR1
Provider selector visible at ALL times (not just for empty conversations).

### Implementation
- Create `LiveProviderSwitcher.tsx` component (or modify `ProviderSelector.tsx`)
- Dropdown always visible in chat header
- Selection change updates state
- Next message uses new provider
- History preserved (useChat messages array unchanged)

### Pattern
```typescript
// State tracks current provider
const [provider, setProvider] = useState('anthropic');

// On provider change, just update state
// Next sendMessage will use new provider
// No need to reset messages
```

---

## API Contract

### POST /api/validate

**Request Body:**
```json
{
  "chartPath": "/path/to/chart",
  "workspaceId": "uuid",
  "revisionNumber": 1
}
```

**Response Body:**
```json
{
  "success": true,
  "result": {
    "chartPath": "/path/to/chart",
    "overallStatus": "warning",
    "stages": [
      {
        "name": "helm-lint",
        "status": "passed",
        "issues": []
      },
      {
        "name": "helm-template", 
        "status": "passed",
        "issues": []
      },
      {
        "name": "kube-score",
        "status": "warning",
        "issues": [
          {
            "severity": "warning",
            "message": "Container has no resource limits",
            "file": "templates/deployment.yaml",
            "line": 25,
            "check": "container-resources"
          }
        ]
      }
    ],
    "totalIssues": 1,
    "errorCount": 0,
    "warningCount": 1
  }
}
```

**Error Response:**
```json
{
  "success": false,
  "message": "Invalid chart path",
  "code": "VALIDATION_ERROR"
}
```

---

## ValidationResults Component

Render validation output with:
- Collapsible sections per stage (lint, template, kube-score)
- Color-coded severity (red=error, yellow=warning, blue=info)
- File location links if available
- Summary counts at top

### Pseudocode
```tsx
function ValidationResults({ result }: { result: ValidationResult }) {
  return (
    <div className="validation-results">
      <SummaryBadges errors={result.errorCount} warnings={result.warningCount} />
      
      {result.stages.map(stage => (
        <CollapsibleStage key={stage.name} stage={stage}>
          {stage.issues.map(issue => (
            <IssueRow key={issue.id} issue={issue} />
          ))}
        </CollapsibleStage>
      ))}
    </div>
  );
}
```

---

## Implementation Order

| Priority | Task | Effort |
|----------|------|--------|
| 1 | `pkg/validation/types.go` | 30 min |
| 2 | `pkg/validation/helm.go` (lint + template) | 1-2 hours |
| 3 | `pkg/validation/kubescore.go` | 1 hour |
| 4 | `pkg/validation/parser.go` | 1 hour |
| 5 | `pkg/validation/pipeline.go` | 1 hour |
| 6 | `pkg/api/handlers/validate.go` | 30 min |
| 7 | Register route in `server.go` | 15 min |
| 8 | `lib/ai/tools/validateChart.ts` | 30 min |
| 9 | Add to `createTools()` | 15 min |
| 10 | `ValidationResults.tsx` | 1 hour |
| 11 | Live provider switcher | 1 hour |
| 12 | Integration testing | 1 hour |

---

## Success Criteria (All Must Pass)

- [ ] "Validate my chart" triggers validateChart tool
- [ ] helm lint executes and results parsed
- [ ] helm template executes and results parsed
- [ ] kube-score executes and results parsed (non-fatal if missing)
- [ ] ValidationResults component renders all severity levels
- [ ] Live provider switching preserves conversation history
- [ ] All PR1 and PR1.5 functionality still works
- [ ] Path validation prevents traversal attacks
- [ ] No command injection vulnerabilities
- [ ] Go tests for validation pipeline

---

## Testing Strategy

### 1. Go Validation Tests
Create `pkg/validation/pipeline_test.go`:
- Test with valid chart directory
- Test with invalid path (should error)
- Test path traversal prevention
- Test partial results on stage failure

### 2. API Endpoint Testing
```bash
curl -X POST http://localhost:8080/api/validate \
  -H "Content-Type: application/json" \
  -d '{"chartPath":"/path/to/chart","workspaceId":"test"}'
```

### 3. Tool Integration Testing
Add tests to existing integration test file: `lib/ai/__tests__/integration/tools.test.ts`

Also test manually:
1. Navigate to `/test-ai-chat`
2. Ask: "Validate my chart"
3. Verify tool invocation and results display

### 4. Live Provider Switch Testing
1. Start conversation with Claude
2. Switch to GPT-4o mid-conversation
3. Verify next message uses new provider
4. Verify history preserved

---

## DO NOT

- Use shell command strings (security risk)
- Skip path validation
- Crash on kube-score failure (graceful degradation)
- Modify PR1/PR1.5 core functionality
- Remove existing tools

---

## COMPLETION REQUIREMENTS

When you have completed ALL success criteria, create:

**File: `docs/PR2_COMPLETION_REPORT.md`**

### Required Sections:

#### 1. Files Summary Table
```markdown
| Action | File Path | Description |
|--------|-----------|-------------|
| Created | pkg/validation/... | ... |
```

#### 2. Validation Pipeline Architecture
- Diagram showing stage flow
- How failures are handled
- Output format

#### 3. Go Code Metrics
- Lines of new Go code
- New vs reused code ratio
- Test coverage

#### 4. Security Measures
- Path validation implementation
- Command execution safety
- Input sanitization

#### 5. Test Results
- Go validation tests
- API endpoint tests
- Integration test results

#### 6. UI Components
- ValidationResults screenshots or description
- LiveProviderSwitcher behavior

#### 7. Final Checklist
- [ ] All 5 tools working (4 from PR1.5 + validateChart)
- [ ] Validation pipeline complete
- [ ] Security measures implemented
- [ ] Live provider switching works
- [ ] All previous functionality preserved

---

*When your completion report is ready, the Master Orchestrator will perform final project review.*

