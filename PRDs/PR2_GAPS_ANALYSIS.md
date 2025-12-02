# PR2 PRD Gap Analysis

**Purpose**: Document identified gaps in PR2 PRDs that require verification via codebase exploration before finalizing specifications.

**Status**: Pending exploration of forked repository

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

**The Problem**:
The architecture shows Next.js API calling Go backend at `/api/validate`, but nowhere do we specify:
- Is Go a separate service on a different port?
- What's the URL? (`GO_BACKEND_URL` referenced but never defined)
- How does this work in local dev vs production?
- Is there authentication between Next.js and Go?
- Do they share a process or are they separate services?

**What We Assume**:
```
Next.js (port 3000?) → HTTP → Go Backend (port 8080?)
```

**What We Need to Discover**:
1. How does the existing Chartsmith architecture connect frontend to Go?
2. Is there an existing pattern we should follow?
3. What environment variables exist for backend URLs?

**Exploration Commands**:
```bash
# Check for existing backend URL configuration
grep -r "BACKEND\|GO_\|API_URL" --include="*.ts" --include="*.tsx" --include=".env*"

# Check how existing API routes call Go
find . -path "*/api/*" -name "*.ts" -exec grep -l "fetch\|http" {} \;

# Look for proxy configuration
cat next.config.js next.config.ts 2>/dev/null | grep -i proxy
```

**Resolution Required**: Document communication pattern in Tech PRD, add `GO_BACKEND_URL` to environment variables section.

---

### Gap 2: Chart Path Resolution

**The Problem**:
When user says "validate my chart", the `validateChart` tool requires `chartPath` as input. But:
- How does the AI know which chart to validate?
- Is there a "current chart in context" concept in Chartsmith?
- Does the user explicitly provide the path, or is it inferred?
- What if multiple charts exist?

**What We Assume**:
There's some existing mechanism that tracks "the chart the user is working on."

**What We Need to Discover**:
1. Does Chartsmith track a "current chart" or "active project"?
2. How do existing chart operations know which chart to modify?
3. Is chart path passed in system prompt context?
4. Is there a file/project selector in the UI?

**Exploration Commands**:
```bash
# Look for chart path/context management
grep -r "chartPath\|currentChart\|activeChart" --include="*.ts" --include="*.tsx"

# Check system prompts for context injection
grep -r "system\|prompt" --include="*.ts" | grep -i chart

# Look for project/file state management
grep -r "useState.*chart\|useContext.*chart" --include="*.tsx"
```

**Resolution Options**:
- A) Document existing mechanism in Tech PRD
- B) Specify AI should ask "Which chart would you like me to validate?" if unknown
- C) Add chart selector component to validation flow

---

### Gap 3: Existing Tools Preservation

**The Problem**:
PR1 Product PRD states:
> "File context tools work as before"
> "Chart generation tools function correctly"

But we never identified what tools currently exist. When PR2 adds `validateChart` to the `tools` object in `streamText`, we must include existing tools too.

**What We Assume**:
Chartsmith has existing tool implementations for file/chart operations.

**What We Need to Discover**:
1. What tools exist today?
2. Where are they defined?
3. What format are they in (custom implementation vs standard)?
4. Do they need migration to AI SDK `tool()` format?

**Exploration Commands**:
```bash
# Look for tool definitions
grep -r "tool\|function\|tools" --include="*.ts" | grep -i "definition\|schema\|execute"

# Look for existing Anthropic tool usage
grep -r "tool_use\|tool_result" --include="*.ts" --include="*.tsx"

# Check for function calling patterns
grep -r "functions\|function_call" --include="*.ts"
```

**Resolution Required**: 
- Inventory existing tools
- Ensure PR2 tool registration includes all existing tools
- Document in Tech PRD which tools are preserved

---

## 🟡 Medium Gaps (Should Address)

### Gap 4: System Prompt for Tool Usage

**The Problem**:
The AI needs guidance on when to invoke `validateChart`. Current PRDs say:
> "AI recognizes validation intent from context"

But there's no system prompt update specified. The tool `description` field alone may not reliably trigger invocation.

**Current State**:
Tool description says:
> "Validate a Helm chart for syntax errors, rendering issues, and Kubernetes best practices. Use when user asks to validate, check, lint, or verify their chart."

**What May Be Missing**:
System prompt instruction like:
> "When the user asks you to validate, check, lint, or verify their chart, use the validateChart tool to run automated validation."

**Exploration Commands**:
```bash
# Find existing system prompts
grep -r "system:" --include="*.ts" -A 10
grep -r "systemPrompt\|SYSTEM_PROMPT" --include="*.ts"
```

**Resolution**: Add system prompt snippet to Tech PRD that teaches AI when to use validation tool.

---

### Gap 5: AI Interpretation Flow After Tool Execution

**The Problem**:
After `validateChart` returns results, the PRDs describe the UX (AI explains issues) but not the mechanism:
- Does the AI automatically interpret results?
- Is this built into Vercel AI SDK behavior?
- Do we need to configure anything?

**What We Assume**:
Vercel AI SDK `streamText` with tools automatically sends tool results back to the model for interpretation.

**Clarification Needed**:
Confirm this is default AI SDK behavior and document explicitly in Tech PRD:
> "When a tool returns results, AI SDK automatically includes the result in the next model turn, allowing the AI to interpret and explain the validation findings."

**Resolution**: Add explicit note in Tech PRD about tool result interpretation flow.

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

**The Problem**:
PRDs reference `test_chart/` but don't specify what test scenarios we need.

**What We Need**:
- Valid chart (should pass all checks)
- Chart with lint errors (missing version, bad YAML)
- Chart with template errors (undefined values)
- Chart with kube-score warnings (no resource limits)

**Exploration Commands**:
```bash
# Check existing test fixtures
ls -la test_chart/ testdata/
cat test_chart/Chart.yaml 2>/dev/null
```

**Resolution**: Add test fixture requirements to PR2 Instructions after exploration confirms what exists.

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

| Gap | Severity | Affected PRD | Resolution Path |
|-----|----------|--------------|-----------------|
| Next.js ↔ Go communication | 🔴 Critical | Tech PRD | Explore, then document |
| Chart path resolution | 🔴 Critical | Tech PRD | Explore existing mechanism |
| Existing tools preservation | 🔴 Critical | Tech PRD + Instructions | Inventory existing tools |
| System prompt for tools | 🟡 Medium | Tech PRD | Add after exploration |
| AI interpretation flow | 🟡 Medium | Tech PRD | Clarify SDK behavior |
| Error message mapping | 🟡 Medium | Tech PRD | Add mapping table |
| Test fixtures | 🟢 Minor | PR2 Instructions | Document after exploration |
| Provider switch during validation | 🟢 Minor | Tech PRD | Add clarifying note |

---

## Exploration Checklist for Different Agent

Before editing PRDs, explore the Chartsmith repo to answer:

### Critical Questions
- [ ] How does Next.js app communicate with Go backend? (ports, URLs, pattern)
- [ ] Is there a "current chart" context mechanism? How does AI know which chart?
- [ ] What tools exist today? Where defined? What format?

### Medium Questions
- [ ] Where are system prompts defined? What do they contain?
- [ ] How do existing tool results get interpreted?

### Minor Questions
- [ ] What test charts exist in `test_chart/` or `testdata/`?
- [ ] What error handling patterns exist?

### Exploration Commands Summary
```bash
# Communication pattern
grep -r "BACKEND\|GO_\|API_URL" --include="*.ts" --include="*.tsx" --include=".env*"
cat next.config.js next.config.ts 2>/dev/null

# Chart context
grep -r "chartPath\|currentChart\|activeChart" --include="*.ts" --include="*.tsx"

# Existing tools
grep -r "tool\|function\|tools" --include="*.ts" | grep -i "definition\|schema"
grep -r "tool_use\|tool_result" --include="*.ts"

# System prompts
grep -r "system:" --include="*.ts" -A 10
grep -r "systemPrompt\|SYSTEM_PROMPT" --include="*.ts"

# Test fixtures
ls -la test_chart/ testdata/
```

---

## Post-Exploration Actions

After exploration, return to this document and:
1. Mark each checkbox as discovered
2. Note findings next to each gap
3. Determine which PRDs need updates
4. Create targeted edits for Tech PRD and Instructions

---

*Document End - PR2 Gap Analysis*
