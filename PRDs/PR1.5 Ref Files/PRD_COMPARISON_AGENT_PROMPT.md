# PRD Comparison and Refinement Analysis Prompt

## Your Role

You are a senior technical reviewer evaluating a set of Product and Technical PRDs for a programming assignment. Your task is to:

1. Compare all PRDs against the source requirements documents
2. Identify any gaps, inconsistencies, or misalignments
3. Evaluate proposed changes and refinements from multiple sources
4. Recommend a final consolidated set of changes
5. Flag any concerns about requirement coverage or architectural soundness

---

## Context

### The Assignment

This is a week-long programming challenge called "The Uncharted Territory Challenge" with the following core constraints:

- Fork a substantial open-source repository (Chartsmith by Replicated)
- Learn a new programming language never used before (Go)
- Build non-trivial features equivalent to 1-2 weeks of team work
- Must be brownfield work extending existing architecture, not greenfield
- Demonstrate understanding of existing system before modifying it

The specific project is migrating Chartsmith from a custom Anthropic SDK implementation to Vercel AI SDK, while adding a new Chart Validation Agent feature.

### PR Structure

The work is divided into three pull requests:

| PR | Name | Purpose |
|----|------|---------|
| PR1 | AI SDK Foundation | Establish new `/api/chat` route with `streamText`, `useChat` hook, provider selection UI |
| PR1.5 | Migration & Feature Parity | Complete tool migration, Go HTTP server for tool execution, system prompt migration, remove `@anthropic-ai/sdk` |
| PR2 | Validation Agent | Add chart validation pipeline (helm lint, template, kube-score), `validateChart` tool, validation results UI |

---

## Documents to Analyze

### Source Requirements (Authoritative)

**Document A: Replicated_Chartsmith.md**

Must Haves:
1. Replace custom chat UI with Vercel AI SDK
2. Migrate from direct `@anthropic-ai/sdk` usage to AI SDK Core
3. Maintain all existing chat functionality (streaming, messages, history)
4. Keep existing system prompts and behavior (user roles, chart context, etc.)
5. All existing features continue to work (tool calling, file context, etc.)
6. Tests pass (or are updated to reflect new implementation)

Nice to Haves:
1. Demonstrate easy provider switching (show how to swap Anthropic for OpenAI)
2. Improve the streaming experience using AI SDK optimizations
3. Simplify state management by leveraging AI SDK's built-in patterns

Submission Requirements:
- Pull Request into replicatedhq/chartsmith repo
- Update ARCHITECTURE.md to reflect new AI SDK integration
- Tests pass or are updated
- Demo video showing: app starting, creating chart via chat, streaming responses, 1-2 key code changes

**Document B: Uncharted_Territory_Challenge.md**

Core Requirements:
- Fork substantial open-source repository
- Learn new programming language + ecosystem (Go in this case)
- Understand existing architecture before modifying
- Build non-trivial feature or transformation (equivalent to 1-2 weeks team work)
- Brownfield work, not greenfield
- Demonstrate brownfield mastery, technical achievement, learning velocity, software quality, ambition & scope

---

### PRDs to Evaluate

You will be provided with:

1. **PR1_Product_PRD.md** - Functional requirements for AI SDK Foundation
2. **PR1_Tech_PRD.md** - Technical specification for AI SDK Foundation
3. **PR1_5_PLAN.md** - High-level plan for Migration & Feature Parity
4. **PR1_5_HIGH_LEVEL_SUMMARY_REVISED.md** - Detailed PR1.5 deliverables
5. **PR2_Product_PRD.md** - Functional requirements for Validation Agent
6. **PR2_Tech_PRD.md** - Technical specification for Validation Agent
7. **PR2_SUPPLEMENTAL_DETAILS.md** - Component pseudocode for PR2
8. **CHARTSMITH_ARCHITECTURE_DECISIONS.md** - ADRs capturing key decisions

---

### Proposed Changes to Evaluate

**Change Set 1: Utility Tool Migration (from initial review)**

Adds two tools that were originally deferred:
- `latestSubchartVersion` - ArtifactHub lookup for subchart versions
- `latestKubernetesVersion` - Current K8s version information

Rationale: Original requirement states "All existing features continue to work (tool calling, file context, etc.)" - these are existing tools that would otherwise be lost.

New files:
- `lib/ai/tools/latestSubchartVersion.ts`
- `lib/ai/tools/latestKubernetesVersion.ts`
- `pkg/api/handlers/versions.go`

**Change Set 2: Error Response Contract (from initial review)**

Adds standardized error handling:
- `ErrorResponse` struct with `success`, `error`, `code`, `details` fields
- HTTP status code mapping
- Shared `callGoEndpoint<T>()` utility for consistent error handling

New files:
- `pkg/api/errors.go`
- `lib/ai/tools/utils.ts`

**Change Set 3: PR1.5 Refinements (from external reviewer)**

Seven refinements:
1. Explicitly declare UI deprecation of legacy chat
2. Add security & auth note for Go tool endpoints
3. Specify error response structure (overlaps with Change Set 2)
4. Clarify reuse of existing workspace logic
5. Add rationale for dropping Go→LLM calls
6. Add success metric about tool invocation
7. Add optional future cleanup section

**Change Set 4: Go Handler Implementation Notes (from external reviewer)**

Detailed implementation guidance for Go handlers:
- CreateChart handler with specific function calls to reuse
- GetChartContext handler with data loading patterns
- UpdateChart handler with patch application approach
- TextEditor handler with existing tool logic reuse
- Common patterns (auth, errors, NOTIFY events)

---

## Your Analysis Tasks

### Task 1: Requirements Gap Analysis

For each source requirement, verify it is addressed:

| Requirement | PR Coverage | Gap? | Severity |
|-------------|-------------|------|----------|
| [requirement text] | [which PR/section] | [Yes/No] | [Critical/Medium/Low] |

Flag any requirements that are:
- Not addressed at all
- Partially addressed
- Addressed but with caveats

### Task 2: Architectural Consistency Check

Evaluate whether the architectural decisions are:
- Internally consistent across all PRDs
- Aligned with the stated "brownfield" requirement
- Justified with clear rationale

Specific questions to answer:
1. Does the "parallel chat system" approach constitute "replacement" as required?
2. Is the "Go makes no LLM calls" decision sound and well-justified?
3. Does the tool migration strategy maintain full feature parity?
4. Is the HTTP-based tool communication appropriate vs. the existing PostgreSQL queue pattern?

### Task 3: Change Set Evaluation

For each proposed change set, evaluate:

| Change Set | Should Incorporate? | Rationale | Priority |
|------------|---------------------|-----------|----------|
| 1: Utility Tools | [Yes/No/Partial] | [why] | [MUST/SHOULD/COULD] |
| 2: Error Contract | [Yes/No/Partial] | [why] | [MUST/SHOULD/COULD] |
| 3: PR1.5 Refinements | [Yes/No/Partial] | [why] | [MUST/SHOULD/COULD] |
| 4: Go Implementation Notes | [Yes/No/Partial] | [why] | [MUST/SHOULD/COULD] |

For any "Partial" recommendations, specify which items to include and which to exclude.

### Task 4: Conflict Resolution

Identify any conflicts between:
- Different change sets
- Change sets and existing PRDs
- Different PRDs

For each conflict, recommend resolution.

### Task 5: Final Recommendations

Provide:
1. **Consolidated change list** - Single authoritative list of all changes to incorporate
2. **Updated success criteria** - Any additions or modifications
3. **Risk flags** - Any concerns about feasibility, scope, or requirement compliance
4. **Priority order** - If time-constrained, what's essential vs. deferrable

---

## Output Format

Structure your response as:

```markdown
# PRD Comparison Analysis Report

## Executive Summary
[2-3 paragraph overview of findings]

## Requirements Gap Analysis
[Task 1 output]

## Architectural Assessment
[Task 2 output]

## Change Set Evaluation
[Task 3 output]

## Conflict Resolution
[Task 4 output]

## Final Recommendations
[Task 5 output]

## Appendix: Consolidated Changes
[Complete list of all recommended changes with file locations]
```

---

## Important Notes

- Be rigorous but pragmatic - this is a week-long assignment, not a production system
- The primary goal is meeting assignment requirements while demonstrating competence
- When in doubt, favor changes that strengthen "brownfield mastery" perception
- Implementation complexity should be weighed against demonstrable value
- The candidate has no prior Go experience; implementation notes should account for this

---

*End of Prompt*
