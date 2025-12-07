# PR4: Chartsmith Validation Agent & Live Provider Switching
## Functional Requirements Document

**Version**: 1.0
**PR**: PR4 (formerly PR2)
**Status**: Ready for Implementation
**Prerequisite**: PR3.0/3.1 merged (AI SDK infrastructure complete)

> **Note**: All file paths in this document are relative to the `chartsmith/` directory.

---

## Executive Summary

### Purpose
Extend Chartsmith with an AI-powered Chart Validation Agent that provides real-time validation feedback, plus complete the provider switching UX to support live mid-conversation model changes.

### Business Value
- **Proactive error prevention**: Catch chart issues before deployment
- **Actionable AI feedback**: Natural language explanations of validation results
- **Best practices enforcement**: kube-score integration ensures K8s standards
- **Flexible model selection**: Switch providers without losing conversation context

### Scope
This PR delivers the Chart Validation Agent feature and completes the provider switching UX, building on the AI SDK infrastructure established in PR3.0/3.1.

---

## Current State (After PR3.1)

### What Already Exists

**AI SDK Chat Route** (`chartsmith-app/app/api/chat/route.ts`):
- `streamText` integration with tool support
- Provider/model parameters accepted in request body
- Buffered and immediate tool execution modes
- Intent classification pre-processing

**Tool Infrastructure**:
- Factory pattern: `createTools()` and `createBufferedTools()`
- Go backend communication: `callGoEndpoint()` utility
- 5 existing tools: `getChartContext`, `textEditor`, `latestSubchartVersion`, `latestKubernetesVersion`, `convertK8sToHelm`

**Go HTTP Server** (`pkg/api/server.go`):
- Running on port 8080
- 8 existing handler files in `pkg/api/handlers/`
- Standard patterns for error responses and realtime updates

**Provider Configuration**:
- `AVAILABLE_PROVIDERS` and `AVAILABLE_MODELS` defined
- `getModel(provider, model)` factory function
- `ProviderSelector` component exists (but not integrated)

### What's Missing

1. **Validation Pipeline**: No validation capability exists
2. **Provider Switching UI**: `ProviderSelector` exists but is unused
3. **Provider State in Chat**: `useAISDKChatAdapter` hardcodes provider/model

---

## Problem Statement

### Current Pain Points
1. **Manual validation burden**: Users run helm lint separately
2. **Cryptic error messages**: CLI output not user-friendly
3. **Missing best practices**: No automated K8s standards checking
4. **Provider lock-in**: Cannot switch models during conversation

### Desired State
- AI validates charts automatically when requested
- Validation results explained in natural language with fix suggestions
- Best practices checked via kube-score integration
- Providers switchable mid-conversation

---

## User Stories

### US-1: On-Demand Chart Validation
**As a** Chartsmith user
**I want** to ask the AI to validate my chart
**So that** I catch errors before deploying

**Acceptance Criteria**:
- User can say "validate my chart" or similar phrases
- AI invokes validation tool automatically
- Results include lint errors, template issues, and best practices
- Clear indication when validation passes vs fails

### US-2: Natural Language Validation Feedback
**As a** Chartsmith user
**I want** validation results explained in plain language
**So that** I understand issues without Kubernetes expertise

**Acceptance Criteria**:
- Each issue explained with context
- Severity clearly indicated (critical/warning/info)
- Specific fix suggestions provided
- Code examples for fixes when appropriate

### US-3: Best Practices Scoring
**As a** Chartsmith user
**I want** to know if my chart follows Kubernetes best practices
**So that** my deployments are secure and reliable

**Acceptance Criteria**:
- kube-score checks executed on rendered templates
- Security issues flagged (missing resource limits, privileged containers)
- Reliability issues flagged (missing probes, anti-affinity)
- Overall score or summary provided

### US-4: Validation Fix Suggestions
**As a** Chartsmith user
**I want** the AI to suggest how to fix validation issues
**So that** I can quickly resolve problems

**Acceptance Criteria**:
- Each issue includes actionable fix suggestion
- AI can offer to apply fixes when requested
- Multiple fix options when alternatives exist
- Explanations of why fix is recommended

### US-5: Live Provider Switching
**As a** Chartsmith user
**I want** to switch AI models during a conversation
**So that** I can try different models without starting over

**Acceptance Criteria**:
- Provider dropdown visible at all times
- Switching preserves conversation history
- New model used for next response only
- Visual indicator of current model

### US-6: Validation Trigger Phrases
**As a** Chartsmith user
**I want** natural ways to trigger validation
**So that** I don't need to learn special commands

**Acceptance Criteria**:
- "Validate my chart" triggers validation
- "Check for errors" triggers validation
- "Is my chart correct?" triggers validation
- "Run helm lint" triggers validation
- AI recognizes validation intent from context

---

## Functional Requirements

### FR-1: Validation Pipeline

#### FR-1.1: Validation Levels
| Level | Tool | Checks |
|-------|------|--------|
| 1 | helm lint | Syntax, structure, best practices |
| 2 | helm template | Template rendering, YAML generation |
| 3 | kube-score | K8s best practices, security, reliability |

#### FR-1.2: Validation Execution
- All three levels run in sequence
- Level 2 only runs if Level 1 passes
- Level 3 only runs if Level 2 succeeds
- Partial results returned if pipeline fails mid-way

#### FR-1.3: Validation Scope
- Validates current chart in workspace context
- Uses current values configuration
- Respects chart dependencies
- Handles subcharts if present

### FR-2: Validation Results

#### FR-2.1: Result Structure
Each validation result includes:
- **Overall Status**: pass / warning / fail
- **Tool Results**: Individual results per validation level
- **Issues List**: All discovered issues with details
- **Summary**: Human-readable summary

#### FR-2.2: Issue Details
Each issue includes:
- **Severity**: critical / warning / info
- **Source**: Which tool found it (lint/template/kube-score)
- **Location**: File and line number when available
- **Message**: Technical description
- **Suggestion**: How to fix

#### FR-2.3: Best Practices Scoring
kube-score results include:
- Numeric score (0-10 scale)
- List of passed checks
- List of failed checks with explanations
- Security-specific findings highlighted

### FR-3: AI Interpretation

#### FR-3.1: Result Explanation
When validation completes, AI should:
- Summarize overall status in first sentence
- List critical issues first
- Explain each issue in plain language
- Provide specific fix suggestions
- Offer to help implement fixes

#### FR-3.2: Contextual Awareness
AI should:
- Reference specific files in user's chart
- Use chart name and values in explanations
- Remember validation context for follow-up questions
- Correlate issues with recent chat history

#### FR-3.3: Fix Assistance
When user asks to fix issues:
- AI proposes specific code changes
- Shows before/after for clarity
- Explains why change fixes the issue
- Offers to validate again after fixes

### FR-4: Live Provider Switching

#### FR-4.1: UI Component
- Dropdown selector always visible in chat header
- Shows current model name/icon
- Expands to show all available providers
- Selection changes take effect immediately

#### FR-4.2: Switching Behavior
- Conversation history preserved completely
- Only future responses use new model
- No confirmation dialog needed
- Visual feedback on switch (brief indicator)

#### FR-4.3: Context Handling
- Full message history sent to new model
- System prompt remains consistent
- Tool definitions unchanged
- No re-initialization needed

### FR-5: Validation Tool Integration

#### FR-5.1: Tool Definition
AI SDK tool following existing factory pattern:
- Factory function: `createValidateChartTool(authHeader, workspaceId, revisionNumber)`
- Clear description for AI to understand when to use
- Input schema for workspace context and options
- Execute function that calls Go backend via `callGoEndpoint()`

#### FR-5.2: Tool Invocation
AI should invoke validation tool when:
- User explicitly requests validation
- User asks about chart correctness
- User asks for error checking
- Context suggests validation needed

#### FR-5.3: Tool Result Rendering
Validation results displayed as:
- Formatted summary card in chat (like PlanChatMessage pattern)
- Expandable details for each issue
- Color-coded severity indicators
- Copy-able code suggestions

---

## Non-Functional Requirements

### NFR-1: Performance
- Validation completes in < 30 seconds for typical charts
- UI remains responsive during validation
- Progress indication for long-running validation
- Timeout handling for stuck processes

### NFR-2: Reliability
- Graceful handling of missing helm/kube-score binaries
- Partial results if one tool fails
- Clear error messages for tool failures
- No crash on malformed chart input

### NFR-3: Security
- Validation runs in isolated context
- No arbitrary code execution
- Chart paths validated/sanitized
- Results don't expose system paths

### NFR-4: Accuracy
- No false positives from tool parsing
- Correct severity classification
- Accurate file/line references
- Valid fix suggestions

---

## UI/UX Specifications

### Validation Results Display

**Location**: Inline in chat as assistant message (following PlanChatMessage pattern)

**Components**:
```
+-----------------------------------------------------------+
| Validation Results                                         |
+-----------------------------------------------------------+
| Overall: 2 warnings, 1 critical issue                     |
+-----------------------------------------------------------+
| CRITICAL: Container missing memory limits                  |
|    deployment.yaml:24                                      |
|    Add resources.limits.memory to container spec           |
|                                                            |
| WARNING: Image uses :latest tag                            |
|    deployment.yaml:18                                      |
|    Pin to specific version (e.g., nginx:1.21.0)           |
|                                                            |
| WARNING: No readiness probe defined                        |
|    deployment.yaml:15                                      |
|    Add readinessProbe for health checking                 |
+-----------------------------------------------------------+
| kube-score: 6/10                                           |
| [Show details] [Validate again]                            |
+-----------------------------------------------------------+
```

**Interactions**:
- Click issue to expand full details
- Click "Show details" for complete kube-score output
- Click "Validate again" to re-run validation

### Live Provider Switcher

**Location**: Chat header, alongside existing persona selector

**Layout**:
```
+-----------------------------------------------------------+
| Chartsmith Chat              [Claude Sonnet v] [Role v]   |
+-----------------------------------------------------------+
|                    +------------------+                    |
|                    | * Claude Sonnet  |                    |
|                    |   GPT-4o         |                    |
|                    |   GPT-4o-mini    |                    |
|                    +------------------+                    |
```

**States**:
- Collapsed: Shows current model name
- Expanded: Shows all available models with checkmark on current
- Switching: Brief loading indicator, then collapsed

**Behavior**:
- Click to expand
- Click option to switch
- Click outside to collapse
- Keyboard navigation supported

### Validation Progress Indicator

**During validation**:
```
+-----------------------------------------------------------+
| Validating chart...                                        |
|    * helm lint complete                                    |
|    * helm template complete                                |
|    Running kube-score...                                   |
+-----------------------------------------------------------+
```

---

## API Contracts

### Validation Endpoint

**Endpoint**: `POST /api/validate`

**Request**:
```json
{
  "workspaceId": "ws_123",
  "revisionNumber": 5,
  "values": { "key": "value" },
  "strictMode": false,
  "kubeVersion": "1.28"
}
```

**Response**:
```json
{
  "validation": {
    "overall_status": "warning",
    "timestamp": "2024-01-15T10:30:00Z",
    "duration_ms": 2500,
    "results": {
      "helm_lint": {
        "status": "pass",
        "issues": []
      },
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
            "resource": "apps/v1/Deployment/my-app",
            "check": "container-resources",
            "message": "Container does not have memory limit",
            "file": "templates/deployment.yaml",
            "line": 24,
            "suggestion": "Add resources.limits.memory"
          }
        ]
      }
    }
  }
}
```

**Error Response**:
```json
{
  "error": "validation_failed",
  "message": "helm lint failed",
  "details": {
    "tool": "helm_lint",
    "exit_code": 1,
    "stderr": "Error: Chart.yaml is missing required field"
  }
}
```

### Chat Endpoint (Existing)

**Endpoint**: `POST /api/chat` (already exists from PR3.0)

The chat endpoint already accepts provider/model parameters. No changes needed to the API contract.

---

## Tool Calling Specification

### Validation Tool Schema

**Name**: `validateChart`

**Description**: "Validate a Helm chart for syntax errors, rendering issues, and Kubernetes best practices. Use this when the user asks to validate, check, lint, or verify their chart."

**Input Schema**:
```
{
  values: object (optional) - Values to use for rendering
  strictMode: boolean (optional) - Fail on warnings
  kubeVersion: string (optional) - Target Kubernetes version
}
```

Note: `workspaceId` and `revisionNumber` are captured by the tool factory from context.

**Output**: Validation result object as defined in API contract

### Tool Invocation Triggers

AI should recognize these patterns:
- "validate my chart"
- "check for errors"
- "run helm lint"
- "is my chart correct?"
- "any issues with the chart?"
- "check best practices"
- "validate before deploying"

---

## Validation Error Categories

### helm lint Errors
| Category | Example | Severity |
|----------|---------|----------|
| Missing required field | Chart.yaml missing version | Critical |
| Invalid YAML syntax | Malformed template | Critical |
| Missing icon | No icon in Chart.yaml | Info |
| API deprecation | Using deprecated API | Warning |

### helm template Errors
| Category | Example | Severity |
|----------|---------|----------|
| Render failure | Template syntax error | Critical |
| Missing value | Required value not set | Critical |
| Invalid reference | .Values.missing.key | Critical |

### kube-score Checks
| Category | Example | Severity |
|----------|---------|----------|
| Resource limits | No memory limit | Critical |
| Security context | Privileged container | Critical |
| Image policy | Using :latest tag | Warning |
| Probes | No readiness probe | Warning |
| Network policy | No NetworkPolicy | Warning |

---

## Success Criteria

### Automated Verification
- [ ] TypeScript compiles: `cd chartsmith-app && npm run build`
- [ ] Go compiles: `cd chartsmith && go build ./...`
- [ ] Linting passes: `cd chartsmith-app && npm run lint`
- [ ] Existing tests pass: `cd chartsmith-app && npm test`

### Manual Verification
- [ ] "Validate my chart" triggers validation tool
- [ ] helm lint, helm template, kube-score all execute
- [ ] Results displayed with severity indicators
- [ ] AI explains issues in natural language
- [ ] Fix suggestions provided for each issue
- [ ] Live provider switching works without losing history
- [ ] All PR3.0/3.1 functionality still works

### Quality Gates
- [ ] Validation completes in < 30 seconds
- [ ] No false positives in result parsing
- [ ] Provider switch feels instant (< 500ms)
- [ ] Error handling for missing tools

---

## Test Requirements

### Validation Tests
| Scenario | Input | Expected |
|----------|-------|----------|
| Valid chart | Well-formed chart | Pass with score |
| Lint error | Missing version | Fail at lint stage |
| Template error | Bad syntax | Fail at template stage |
| Best practice warning | No resource limits | Warning with suggestion |
| All pass | Perfect chart | Pass, high score |

### Provider Switching Tests
| Scenario | Steps | Expected |
|----------|-------|----------|
| Basic switch | Select Claude during chat | Next response from Claude |
| History preserved | Switch, ask follow-up | Context maintained |
| Multiple switches | Switch 3 times | All work correctly |
| Switch during stream | Switch while streaming | Current completes, next uses new |

---

## Dependencies

### External Dependencies
- helm CLI (v3.x)
- kube-score CLI (v1.16+)

### Internal Dependencies (Already Exist)
- Go HTTP server (`pkg/api/server.go`)
- Tool factory pattern (`lib/ai/tools/index.ts`)
- `callGoEndpoint()` utility (`lib/ai/tools/utils.ts`)
- Provider configuration (`lib/ai/provider.ts`)

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Tool not installed | Medium | High | Check on startup, clear error |
| Slow validation | Medium | Medium | Progress indicator, timeout |
| Parsing errors | Medium | Medium | Robust parsing, raw fallback |
| Context loss on switch | Low | High | Preserve full message array |

---

## Out of Scope (PR4)

- Values schema validation (future)
- Environment drift detection (future)
- Dependency vulnerability scanning (future)
- Conversation persistence
- Custom validation rules
- CI/CD integration
- Auto-validate on chart change

---

## Future Enhancements

Post-PR4 features for consideration:
1. **Auto-validate on change**: Validate when chart modified
2. **Inline fix application**: AI applies fixes directly
3. **Validation history**: Track improvements over time
4. **Custom rules**: User-defined validation rules
5. **Security scanning**: Trivy/Checkov integration

---

*Document End - PR4: Functional Requirements*
