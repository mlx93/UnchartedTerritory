# PR2 Gap Analysis Research Prompt

## Command
```
/research_codebase
```

## Prompt

```
Research the Chartsmith codebase to answer the gap analysis questions for PR2 (Chart Validation Agent).

**Prerequisite**: PR1 (Vercel AI SDK migration) is complete or in progress.

## Critical Gaps to Resolve

### Gap 1: Next.js ↔ Go Backend Communication
- How does the Next.js app communicate with the Go backend today?
- What ports are used? Is there a proxy configuration?
- What environment variables control backend URLs?
- Is there authentication between the services?
- How does this work in local dev vs production?

### Gap 2: Chart Path Resolution
- Is there a "current chart" or "active project" concept?
- How do existing chart operations know which chart to modify?
- Is chart path passed in system prompt context?
- How would the AI know which chart to validate?

### Gap 3: Existing Tools Inventory
- What tools/functions are currently defined for the LLM?
- Where are they defined and in what format?
- Do they need migration to AI SDK `tool()` format?
- What file context tools exist?

## Medium Gaps to Resolve

### Gap 4: System Prompts
- Where are system prompts defined?
- What context is injected into prompts?
- How would we add guidance for using the validateChart tool?

### Gap 5: Tool Result Flow
- How do tool results currently get back to the LLM?
- Is there existing tool result rendering in the UI?

### Gap 6: Test Fixtures
- What exists in `test_chart/` or `testdata/`?
- Are there valid and invalid chart examples?

## Context

We're adding a Chart Validation Agent that:
- Adds a `validateChart` tool to the AI
- Calls a new Go backend endpoint `/api/validate`
- Runs helm lint → helm template → kube-score pipeline
- Displays results in chat with AI interpretation

## Reference Documents

Read these files in `docs/prds/` for full specifications:
- `PR2_Product_PRD.md` - Functional requirements
- `PR2_Tech_PRD.md` - Technical specification  
- `PR2_GAPS_ANALYSIS.md` - Full gap analysis with exploration commands
- `PR2_INSTRUCTIONS.md` - Implementation steps
- `PR2_SUPPLEMENTAL_DETAILS.md` - Component pseudocode

## Output

Update `docs/prds/PR2_GAPS_ANALYSIS.md` with:
- ✅/⚠️/❌ status for each gap
- Findings with `file:line` references
- Recommended PRD updates based on discoveries

Also create `docs/research/PR2_CODEBASE_FINDINGS.md` with:
- Architecture diagram showing Next.js ↔ Go communication
- Inventory of existing tools
- Chart context mechanism explanation
- Any risks or blockers identified
```
