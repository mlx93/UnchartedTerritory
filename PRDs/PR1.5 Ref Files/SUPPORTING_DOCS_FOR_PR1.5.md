# Supporting Documents for PR1.5 PRD Creation

## Key Architecture Decision

**Go does NOT need to make LLM calls after PR1.5.**

This simplifies the scope significantly:
- ❌ No AI Gateway endpoint needed
- ❌ No Go LLM client modifications needed
- ✅ Go just exposes HTTP endpoints for tool execution
- ✅ All "thinking" lives in AI SDK Core (Node)

## Documents to Include in New Session

### Priority 1: Essential (Must Include)

| Document | Purpose | Key Sections to Reference |
|----------|---------|---------------------------|
| `PR1.5_HIGH_LEVEL_SUMMARY_REVISED.md` | Blueprint for PRD creation (USE THIS ONE) | All - this is your roadmap |
| `UPDATED_PR_PLAN.md` | Overall architecture and PR relationships | AI Gateway Architecture, PR1.5 section |
| `Replicated_Chartsmith.md` | Original requirements (source of truth) | Success Criteria, Technical Requirements |
| `Uncharted_Territory_Challenge.md` | Challenge constraints | Core Objective, Evaluation Criteria |

### Priority 2: Context (Highly Recommended)

| Document | Purpose | Key Sections to Reference |
|----------|---------|---------------------------|
| `PR1_Product_PRD.md` | What PR1 delivers (PR1.5 builds on this) | User Stories, Functional Requirements |
| `PR1_Tech_PRD.md` | Technical foundation PR1.5 extends | System Architecture, File Structure |
| `PR2_Product_PRD.md` | What PR1.5 must enable | Validation Tool Integration, Tool Schema |
| `PR2_Tech_PRD.md` | PR2's technical needs from PR1.5 | Go Backend Specification |

### Priority 3: Reference (Include if Context Window Allows)

| Document | Purpose | Key Sections |
|----------|---------|--------------|
| `PR2_SUPPLEMENTAL_DETAILS.md` | Component pseudocode patterns | Tool Definition example, Result rendering |
| `ARCHITECTURE_EVALUATION_AND_FEEDBACK.md` | Historical context on why we're doing this | Gap Areas, Trade-offs |

---

## Key Information to Extract Before Session

### From Existing Chartsmith Codebase (if available)

You'll want to reference or summarize:

1. **Current Go LLM client location and usage**
   - `pkg/llm/client.go` - How Anthropic client is created
   - `pkg/llm/execute-action.go` - Where LLM calls happen
   - What functions need modification

2. **Current system prompts**
   - `pkg/llm/system.go` - Prompt content to migrate
   - Role definitions, context injection patterns

3. **Existing tool definitions**
   - `text_editor_20241022` schema
   - `latest_subchart_version` schema
   - `latest_kubernetes_version` schema

4. **Workspace/Chart data models**
   - What fields exist on workspace/chart objects
   - What the "create chart" flow currently does

5. **Database schema relevant to charts**
   - Tables involved in chart creation
   - What pg_notify events are triggered

### From Your Research Documents

If you have these from earlier analysis, include summaries:
- `CURRENT_STATE_ANALYSIS.md` - Existing architecture details
- `VERCEL_AI_SDK_REFERENCE.md` - SDK patterns
- Any gap analysis documents

---

## Prompt Structure for New Session

When starting the PR1.5 PRD creation session, structure your prompt like:

```
I need to create detailed Product and Technical PRDs for PR1.5 of my Chartsmith 
AI SDK migration project.

## Context
- PR1 establishes AI SDK chat foundation (useChat, /api/chat, streaming)
- PR1.5 must achieve feature parity and add AI Gateway
- PR2 will add validation agent (depends on PR1.5 infrastructure)

## What PR1.5 Must Deliver
[Paste PR1.5_HIGH_LEVEL_SUMMARY.md content]

## Original Requirements
[Reference key quotes from Replicated_Chartsmith.md]

## Technical Foundation from PR1
[Reference relevant sections from PR1 Tech PRD]

## What PR2 Needs from PR1.5
[Reference PR2's tool integration requirements]

Please create:
1. PR1.5 Product PRD (user stories, functional requirements, acceptance criteria)
2. PR1.5 Technical PRD (architecture, implementation details, file specifications)
```

---

## Questions the New Session Should Answer

The PR1.5 PRDs should clearly address:

### Product PRD Questions
- [ ] What user stories does PR1.5 enable?
- [ ] What is the acceptance criteria for "feature parity"?
- [ ] How does the user experience change after PR1.5?
- [ ] What's the deprecation UX for legacy chat?

### Technical PRD Questions
- [ ] Exact API contracts for Go tool endpoints (charts, editor)
- [ ] How are system prompts structured for tool invocation?
- [ ] What's the request/response flow for "create chart"?
- [ ] How do we handle errors from Go endpoints?
- [ ] What's the shared llmClient.ts interface?
- [ ] How are tools registered in /api/chat?

### Integration Questions
- [ ] How does createChart tool interact with existing workspace creation?
- [ ] What pg_notify events need to fire for Centrifugo updates?
- [ ] How does the new chat integrate with existing workspace state?
- [ ] How do we verify the full flow works (integration test)?

### Go-Specific Questions (Simpler Now!)
- [ ] How does pkg/api/server.go integrate with existing main.go?
- [ ] What existing workspace/chart logic can handlers reuse?
- [ ] How do handlers connect to the database?
- [ ] What's the error response format?

---

## Checklist Before Starting New Session

- [ ] PR1.5_HIGH_LEVEL_SUMMARY_REVISED.md ready (USE THIS VERSION)
- [ ] UPDATED_PR_PLAN.md ready (for overall context, but note gateway is simplified)
- [ ] Original requirements docs accessible
- [ ] PR1 PRDs accessible for reference
- [ ] PR2 PRDs accessible for "what we need to enable"
- [ ] Any codebase analysis notes summarized
- [ ] Clear understanding of existing workspace/chart creation logic in Go

## What Changed from Original Plan

The "Go doesn't need LLM calls" decision removes:
- ❌ `pkg/llm/gateway_client.go`
- ❌ Modifications to `pkg/llm/client.go`
- ❌ Modifications to `pkg/llm/execute-action.go`
- ❌ `/api/ai/llm` gateway endpoint

This reduces Go scope to:
- ✅ Stand up HTTP server (`pkg/api/server.go`)
- ✅ Implement tool handlers (`pkg/api/handlers/*.go`)
- ✅ Connect handlers to existing DB/workspace logic

Much simpler! Go becomes pure application logic, no LLM orchestration.
