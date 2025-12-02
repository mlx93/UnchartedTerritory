---
date: 2025-12-02T05:22:13Z
researcher: mylessjs
git_commit: e615830f7b0b60ecb817d762b9b4e4186f855d85
branch: main
repository: UnchartedTerritory
topic: "PR2 Tool Support Requirements - Moving Tools from Post-PR2 to PR2"
tags: [research, prd-changes, pr2, tools, architecture, ai-sdk]
status: complete
last_updated: 2025-12-02
last_updated_by: mylessjs
---

# Research: PR2 Tool Support Requirements

**Date**: 2025-12-02T05:22:13Z
**Researcher**: mylessjs
**Git Commit**: e615830f7b0b60ecb817d762b9b4e4186f855d85
**Branch**: main
**Repository**: UnchartedTerritory

## Research Question

Analyze PR1/PR2 PRDs to determine what changes are needed to:
1. Support tools in the NEW chat (AI SDK-based) in PR2 (not Post-PR2)
2. Include Go backend API adjustments in PR2 for the validation tool
3. Ensure PR2 PRDs properly build on the expected PR1 end state

## Executive Summary

**Key Finding**: PR1_Tech_PRD.md explicitly states tool support is "Future Enhancement (Post-PR2)" but the user's intent is for PR2 to include tool support in the NEW AI SDK chat.

**Required Changes**:
1. **PR1_Tech_PRD.md**: Revise "Tool Strategy for PR1" section to clarify that PR2 WILL add tools
2. **PR2_Tech_PRD.md**: Move tool support from implied "future" to explicit PR2 scope
3. **PR2_ARCHITECTURE_DECISION_REQUIRED.md**: Confirm Option A (HTTP endpoint) as the decision
4. **PR2 PRDs**: Update to show validateChart tool as first AI SDK tool in NEW chat

---

## Detailed Findings

### Finding 1: Tool Support Explicitly Deferred to Post-PR2

**Location**: `PRDs/PR1_Tech_PRD.md:407-427`

**Current Text**:
```markdown
### Tool Strategy for PR1

**PR1 Scope**: The NEW `/api/chat` route will be **conversational only** (no tools).

**Rationale**:
- Existing Go tools remain in Go backend
- PR2 adds the validation tool which will establish the Go HTTP API pattern
- Full tool migration is a future enhancement, not PR1 scope

**Future Enhancement** (Post-PR2):
If tool support is needed in the new chat, options include:
1. Add HTTP endpoints to Go for tool execution
2. Call Go endpoints from AI SDK tool execute() functions
3. Mirror simple tools in TypeScript
```

**Problem**: The text says "Future Enhancement (Post-PR2)" but user intent is for PR2 to include tool support.

**Required Change**:
- Change "Future Enhancement (Post-PR2)" to "PR2 Implementation"
- Clarify that PR2 WILL add the validateChart tool to the NEW `/api/chat` route
- Remove "post-PR2" language

---

### Finding 2: Architecture Decision Needed for Go HTTP Endpoint

**Location**: `PRDs/PR2_ARCHITECTURE_DECISION_REQUIRED.md`

**Current State**: Decision documented as "NEEDED BEFORE IMPLEMENTATION" with three options:
- **Option A**: Add HTTP endpoint to Go (recommended)
- **Option B**: Use PostgreSQL queue pattern (async)
- **Option C**: Next.js spawns Go CLI

**User Intent**: "API adjustments should be in PR2 (validation tool)"

**Required Decision**: **Option A - Add HTTP Endpoint to Go**

**Rationale**:
- Synchronous response required for AI SDK tool `execute()` function
- Establishes pattern for future tools
- Simplest integration with Vercel AI SDK tools
- User explicitly mentioned "API adjustments" which implies HTTP

**Required Changes**:
1. Update `PR2_ARCHITECTURE_DECISION_REQUIRED.md` to show Option A as DECIDED
2. Update `PR2_Tech_PRD.md` to remove conditional language ("depends on architecture decision")
3. Add `GO_BACKEND_URL` environment variable to PR2 specs

---

### Finding 3: PR2 Tech PRD Has Conditional Tool Implementation

**Location**: `PRDs/PR2_Tech_PRD.md:423-482`

**Current Text** (scattered throughout):
```markdown
**NOTE**: Tool implementation approach depends on architecture decision.
See `PR2_ARCHITECTURE_DECISION_REQUIRED.md`.
```

**Problem**: The PRD hedges on implementation because no decision has been made.

**Required Changes**:
1. Remove conditional language throughout PR2_Tech_PRD.md
2. Commit to Option A (HTTP endpoint) implementation
3. Update tool definition to use confirmed approach:
   ```typescript
   execute: async (input) => {
     const response = await fetch(`${process.env.GO_BACKEND_URL}/api/validate`, {
       method: 'POST',
       headers: { 'Content-Type': 'application/json' },
       body: JSON.stringify(input),
     });
     if (!response.ok) throw new Error(`Validation failed: ${response.statusText}`);
     return response.json();
   }
   ```

---

### Finding 4: PR2 Product PRD References Tool Support Correctly

**Location**: `PRDs/PR2_Product_PRD.md:242-265`

**Current Text**:
```markdown
#### FR-5.1: Tool Definition
AI SDK tool with:
- Clear description for AI to understand when to use
- Input schema for chart path and options
- Output schema for structured results
- Execute function that calls Go backend (via HTTP, queue, or CLI depending on architecture decision)
```

**Assessment**: This is correct but needs the conditional language removed.

**Required Change**: Update FR-5.1 to state:
```markdown
- Execute function that calls Go backend via HTTP POST to /api/validate
```

---

### Finding 5: PR2 Supplemental Details Shows Correct Tool Pattern

**Location**: `PRDs/PR2_SUPPLEMENTAL_DETAILS.md:193-267`

**Current Implementation**:
```typescript
// Option A: HTTP endpoint (assumes Go has HTTP server)
execute: ASYNC (input) => {
  response = await fetch(`${process.env.GO_BACKEND_URL}/api/validate`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(input)
  })
```

**Assessment**: This is the correct implementation for Option A.

**Required Change**: Remove the commented alternative options (B and C).

---

## Expected Post-PR1 State (Starting Point for PR2)

Based on PR1 PRDs, after PR1 is merged:

### New Components Created by PR1:
| Component | Purpose |
|-----------|---------|
| `/api/chat/route.ts` | NEW chat endpoint using `streamText` |
| `lib/ai/provider.ts` | Provider factory for OpenRouter |
| `lib/ai/models.ts` | Model definitions |
| `lib/ai/config.ts` | AI configuration |
| `components/chat/ProviderSelector.tsx` | Model selection UI |
| Chat components | Using `useChat` hook |

### PR1 Chat Route State:
```typescript
// /api/chat/route.ts after PR1
const result = await streamText({
  model: getModel(provider),
  messages,
  system: systemPrompt,
  // NO tools - conversational only
});
return result.toDataStreamResponse();
```

### What PR2 Adds to This:
```typescript
// /api/chat/route.ts after PR2
import { validateChart } from '@/lib/ai/tools/validateChart';

const result = await streamText({
  model: getModel(provider),
  messages,
  system: systemPrompt,
  tools: {
    validateChart  // NEW - added by PR2
  }
});
return result.toDataStreamResponse();
```

---

## Required PRD Changes Summary

### 1. PR1_Tech_PRD.md Changes

**Section**: "Tool Strategy for PR1" (lines 407-427)

**Current**:
```markdown
**Future Enhancement** (Post-PR2):
If tool support is needed in the new chat, options include:
```

**Change To**:
```markdown
**PR2 Implementation**:
Tool support will be added in PR2:
1. PR2 adds HTTP endpoint to Go for validation
2. validateChart tool uses AI SDK tool() format
3. Tool calls Go endpoint from execute() function
4. This establishes the pattern for additional tools
```

---

### 2. PR2_ARCHITECTURE_DECISION_REQUIRED.md Changes

**Section**: Title and status

**Current**:
```markdown
**Status**: DECISION NEEDED BEFORE IMPLEMENTATION
```

**Change To**:
```markdown
**Status**: DECIDED - Option A (HTTP Endpoint)
**Decision Date**: 2025-12-02
**Rationale**:
- Synchronous response required for AI SDK tool execute()
- User confirmed "API adjustments should be in PR2"
- Simplest integration path for tool calling
```

---

### 3. PR2_Tech_PRD.md Changes

**Multiple sections need updates:**

1. **Line 8-9** - Remove "BLOCKED" status:
   ```markdown
   **Status**: Ready for Implementation (Architecture Decided)
   ```

2. **Lines 12-26** - Update architecture decision section:
   ```markdown
   ## Architecture Decision: RESOLVED

   **Decision**: Option A - Add HTTP Endpoint to Go

   This means:
   - Go backend gets a new HTTP server (pkg/api/)
   - /api/validate endpoint handles validation requests
   - Tool execute() calls Go directly via HTTP
   ```

3. **Lines 423-482** - Update tool definition to remove conditionals:
   Remove Options B and C, keep only Option A implementation.

4. **Add new environment variable**:
   ```markdown
   ### New Environment Variables (PR2)

   ```env
   # Go Backend URL for tool calls
   GO_BACKEND_URL=http://localhost:8080
   ```
   ```

---

### 4. PR2_Product_PRD.md Changes

**Section**: FR-5.1 Tool Definition (lines 245-250)

**Change**:
```markdown
#### FR-5.1: Tool Definition
AI SDK tool with:
- Clear description for AI to understand when to use
- Input schema for chart path and options
- Output schema for structured results
- Execute function that calls Go backend via HTTP POST to /api/validate
```

---

### 5. PR2_SUPPLEMENTAL_DETAILS.md Changes

**Section**: validateChart Tool Definition (lines 193-251)

**Remove** the commented Option B and Option C alternatives. Keep only the Option A implementation.

---

## Architecture Decision Confirmation

Based on user requirements and codebase analysis:

### Confirmed: Option A - Add HTTP Endpoint to Go

**What This Means**:

1. **Go Backend Changes (PR2)**:
   - Add `cmd/http.go` or modify `cmd/run.go` to start HTTP server
   - Create `pkg/api/validate.go` with HTTP handler
   - HTTP server runs alongside PostgreSQL listener
   - New port (e.g., 8080) for HTTP API

2. **New Go Package Structure**:
   ```
   pkg/
   ├── api/                     # NEW - HTTP handlers
   │   └── validate.go          # POST /api/validate handler
   ├── validation/              # NEW - Validation logic
   │   ├── pipeline.go          # Orchestration
   │   ├── helm.go              # helm lint/template
   │   ├── kubescore.go         # kube-score
   │   └── types.go             # Type definitions
   └── (existing packages)
   ```

3. **Frontend Tool Call**:
   ```typescript
   execute: async (input) => {
     const response = await fetch(`${process.env.GO_BACKEND_URL}/api/validate`, {
       method: 'POST',
       headers: { 'Content-Type': 'application/json' },
       body: JSON.stringify(input),
     });
     return response.json();
   }
   ```

4. **Environment Variables**:
   ```env
   GO_BACKEND_URL=http://localhost:8080
   ```

---

## Code References

### PR1 Creates These Files:
- `chartsmith-app/app/api/chat/route.ts` - NEW chat endpoint
- `chartsmith-app/lib/ai/provider.ts` - Provider factory
- `chartsmith-app/components/chat/ProviderSelector.tsx` - UI

### PR2 Creates These Files:
- `chartsmith-app/lib/ai/tools/validateChart.ts` - Tool definition
- `chartsmith-app/components/chat/ValidationResults.tsx` - Results UI
- `chartsmith-app/components/chat/LiveProviderSwitcher.tsx` - Always-visible switcher
- `pkg/api/validate.go` - Go HTTP handler
- `pkg/validation/*.go` - Validation pipeline

### Key PRD Sections:
- `PRDs/PR1_Tech_PRD.md:407-427` - Tool strategy (needs update)
- `PRDs/PR2_Tech_PRD.md:12-26` - Architecture decision (needs update)
- `PRDs/PR2_ARCHITECTURE_DECISION_REQUIRED.md` - Full decision doc (needs status update)

---

## Open Questions

1. **Port for Go HTTP server**: Should it be 8080 or another port?
2. **Authentication between Next.js and Go HTTP**: Should there be internal API keys?
3. **Should validation tool be added to existing Go tools list or only AI SDK chat?**: Recommendation is AI SDK chat only for now.

---

## Related Research

- `docs/research/2025-12-02-PRD-FALSE-ASSUMPTIONS-ANALYSIS.md` - Full PRD analysis
- `PRDs/PR2_GAPS_ANALYSIS.md` - Identified gaps
- `PRDs/PR2_ARCHITECTURE_DECISION_REQUIRED.md` - Decision document

---

*Research completed 2025-12-02*
