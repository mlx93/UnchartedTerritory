# Research: Option A (Full Migration) vs Parallel System Evaluation

---
date: 2025-12-01T22:54:15-0600
researcher: Claude Code
git_commit: e615830f7b0b60ecb817d762b9b4e4186f855d85
branch: main
repository: UnchartedTerritory/chartsmith
topic: "Evaluating Migration Approaches Against Replicated_Chartsmith.md Requirements"
tags: [research, migration, PR1, chartsmith, vercel-ai-sdk, architecture-decision]
status: complete
last_updated: 2025-12-01
last_updated_by: Claude Code
---

## Executive Summary

This research evaluates whether **Option A (Full Migration)** or the **Alternative Parallel System Approach** best satisfies the requirements in `Replicated_Chartsmith.md`.

### Recommendation: Parallel System Approach

**The Parallel System approach (current PR1 PRDs) is the correct path forward** because:
1. It explicitly satisfies the requirement "The existing Go backend will remain"
2. It meets all Must-Have success criteria
3. It fits the 3-day PR1 timeline
4. It is lower risk and sets up PR2 cleanly
5. Option A goes significantly beyond the stated requirements

---

## Part 1: Requirement Analysis

### Critical Requirement Statement

**Replicated_Chartsmith.md, Lines 42-45:**
```
Technical Requirements:
- Frontend: TypeScript, Next.js, React
- Backend: The existing Go backend will remain, but may need API adjustments
- AI SDK: Vercel AI SDK (both UI and Core libraries)
- LLM Provider: Any
```

**Key Phrase:** "The existing Go backend will remain"

This explicitly states the Go backend should NOT be replaced. It may need "API adjustments" but remains functional.

### What Needs Solving (Lines 23-26)

1. **"Simplify the frontend"**: Replace custom chat components with Vercel AI SDK UI hooks
2. **"Modernize the backend"**: Replace direct Anthropic SDK calls with AI SDK Core's unified API

### Interpreting "Modernize the backend"

The word "backend" here is ambiguous. It could mean:
- **(A)** The Go backend - All LLM orchestration moves to Next.js
- **(B)** The Next.js API routes - LLM calls in Next.js use AI SDK Core

**Resolution:** Given the explicit statement "Go backend will remain," interpretation (B) is correct. The "backend" being modernized is the Next.js server-side code (API routes), not the Go service.

---

## Part 2: Requirements Mapping

### Must-Have Criteria Analysis

| # | Requirement | Option A (Full Migration) | Parallel System | Assessment |
|---|-------------|---------------------------|-----------------|------------|
| 1 | Replace custom chat UI with Vercel AI SDK | YES - New components | YES - New components | Both satisfy |
| 2 | Migrate from `@anthropic-ai/sdk` to AI SDK Core | YES - All calls migrate | PARTIAL - Frontend only | See below |
| 3 | Maintain all existing chat functionality | YES - But risky rewrite | YES - Existing preserved | Parallel safer |
| 4 | Keep existing system prompts and behavior | YES - Ported to TS | YES - Unchanged in Go | Parallel simpler |
| 5 | All existing features work (tools, file context) | YES - Via Go HTTP | YES - Go unchanged | Both satisfy |
| 6 | Tests pass | RISK - Major changes | YES - Existing tests work | Parallel safer |

### Requirement #2 Deep Dive

**Frontend Anthropic SDK Status:**
- Location: `chartsmith-app/lib/llm/prompt-type.ts`
- Status: **ORPHANED CODE** - not imported or used anywhere
- Impact: Can be safely deleted

**Go Anthropic SDK Status:**
- Location: `chartsmith/pkg/llm/*.go` (11 files)
- Status: **Actively used** for all LLM orchestration
- Impact: Would require major rewrite in Option A

**Conclusion for Requirement #2:**
- The frontend `@anthropic-ai/sdk` is dead code - deleting it satisfies "migrate from"
- The Go SDK is explicitly allowed to stay per "Go backend will remain"
- Adding AI SDK Core to Next.js API routes fulfills "modernize the backend"

---

## Part 3: Approach Comparison

### Option A: Full Migration

**What it proposes:**
1. Move ALL LLM orchestration from Go to Next.js
2. Create new `/api/chat` route with `streamText`
3. Add HTTP server to Go for tool execution only
4. Delete Go LLM code after migration
5. Port system prompts to TypeScript

**Pros:**
- Unified LLM layer in Next.js
- Easier multi-provider support
- Modern AI SDK patterns throughout

**Cons:**
- Contradicts "Go backend will remain"
- 5-6 weeks estimated timeline (vs 3 days for PR1)
- High risk of breaking existing functionality
- Major test rewrite required
- Goes beyond stated requirements

### Parallel System Approach (Current PR1 PRDs)

**What it proposes:**
1. Add NEW `/api/chat` route using Vercel AI SDK
2. Create NEW chat components using `useChat` hook
3. Keep Go backend completely unchanged
4. Two chat systems coexist
5. Delete orphaned `prompt-type.ts`

**Pros:**
- Explicitly follows "Go backend will remain"
- Fits 3-day timeline
- Low risk - existing code unchanged
- Existing tests pass
- Sets up PR2 cleanly

**Cons:**
- Two parallel systems to maintain (temporarily)
- New chat may have limited tool support initially

---

## Part 4: Timeline Reality Check

### Replicated_Chartsmith.md Timeline

The challenge structure implies:
- **PR1**: Days 1-3 (3 days) - AI SDK Migration
- **PR2**: Days 4-6 (3 days) - Validation Agent + Provider Switching

### Option A Timeline (from research doc)

```
Phase 1: Foundation (Week 1-2)
Phase 2: Chat Migration (Week 2-3)
Phase 3: Execute Plan Migration (Week 3-4)
Phase 4: Cleanup & Integration (Week 4-5)
Phase 5: K8s Conversion (Week 5-6)
```

**5-6 weeks of work vs 3 days for PR1**

This is a fundamental mismatch. Option A describes a complete architectural overhaul, not a 3-day migration.

### Parallel System Timeline

```
Day 1: Provider factory + API route
Day 2: Chat components + useChat migration
Day 3: Testing + documentation
```

**This fits the 3-day timeline.**

---

## Part 5: PR2 Implications

### How PR2 is Affected by PR1 Approach

| Aspect | If Option A | If Parallel System |
|--------|-------------|-------------------|
| Go HTTP Server | Already added | Must add in PR2 |
| Validation Tool | Uses existing pattern | Must define communication |
| Live Provider Switching | Natural fit | Natural fit |
| Test Complexity | Major changes | Incremental additions |

### Architecture Decision for PR2

The PR2 PRDs note: "The Go backend currently has NO HTTP server."

**Options for PR2 validation tool:**
- **Option A** (HTTP): Add HTTP endpoint to Go
- **Option B** (Queue): Use PostgreSQL queue pattern
- **Option C** (CLI): Next.js spawns Go CLI

The Parallel System approach sets up PR2 cleanly - PR2 adds the Go HTTP server for validation, which Option A would have added in PR1.

---

## Part 6: Gap Analysis

### What Parallel System Needs to Address

1. **Tool Support in New Chat**
   - Current: Go has 3 tools (text_editor, subchart_version, kubernetes_version)
   - Question: Does new AI SDK chat need tool access?
   - Options:
     a. No tools initially (conversational only)
     b. Add Go HTTP API (pulls PR2 work into PR1)
     c. Simple tools in Next.js (no file editing)

   **Recommendation:** Option (a) for PR1, add validation tool in PR2

2. **Orphaned Code Cleanup**
   - `lib/llm/prompt-type.ts` - Should be deleted
   - `@anthropic-ai/sdk` in package.json - Should be removed

3. **Documentation Updates**
   - ARCHITECTURE.md needs updating
   - New component patterns documented

---

## Part 7: Final Recommendation

### Use the Parallel System Approach

**Rationale:**

1. **Requirement Compliance**
   - "Go backend will remain" - SATISFIED
   - "Replace custom chat UI" - SATISFIED (new components)
   - "AI SDK Core unified API" - SATISFIED (new `/api/chat`)

2. **Risk Management**
   - Existing functionality preserved
   - Incremental change
   - Easy rollback if needed

3. **Timeline Fit**
   - 3 days for PR1 - ACHIEVABLE
   - PR2 builds naturally on PR1

4. **Scope Appropriateness**
   - Fulfills requirements without over-engineering
   - Option A is a future enhancement, not PR1

### PR1 Implementation Path

```
Day 1:
├── Delete orphaned lib/llm/prompt-type.ts
├── Remove @anthropic-ai/sdk from package.json
├── Add AI SDK + OpenRouter dependencies
├── Create lib/ai/provider.ts
├── Create /api/chat/route.ts with streamText

Day 2:
├── Create components/chat/ProviderSelector.tsx
├── Create new Chat component using useChat
├── Update message rendering for parts
├── Wire up provider selection

Day 3:
├── Update tests
├── Update ARCHITECTURE.md
├── Demo video recording
├── PR preparation
```

### PR2 Implementation Path

```
Day 4:
├── Add HTTP server to Go
├── Implement /api/tools/validate endpoint
├── Create validateChart tool definition

Day 5:
├── ValidationResults component
├── LiveProviderSwitcher component
├── Integration testing

Day 6:
├── Polish and documentation
├── PR preparation
```

---

## Appendix: Requirements Checklist

### Must-Have Requirements - Parallel System Approach

| Requirement | How Satisfied |
|-------------|---------------|
| Replace custom chat UI with Vercel AI SDK | New components use `useChat` hook |
| Migrate from @anthropic-ai/sdk to AI SDK Core | Delete orphaned TS code, new API uses AI SDK |
| Maintain existing chat functionality | Go backend unchanged, existing flow works |
| Keep existing system prompts | System prompts remain in Go |
| All existing features work | Go tools continue working |
| Tests pass | Existing tests unchanged, new tests for new code |

### Nice-to-Have Requirements

| Requirement | How Satisfied |
|-------------|---------------|
| Demonstrate provider switching | ProviderSelector component |
| Improve streaming experience | AI SDK Data Stream |
| Simplify state management | `useChat` replaces custom hooks |

---

## Conclusion

**Option A (Full Migration) is over-engineered for PR1.**

While it's a valid future architecture, it:
- Contradicts "Go backend will remain"
- Requires 5-6 weeks vs 3 days
- Adds unnecessary risk

**The Parallel System approach is correct** because it:
- Literally follows the requirements
- Fits the timeline
- Is achievable with low risk
- Sets up PR2 properly

Proceed with the Parallel System approach as documented in the current PR1 PRDs, with the corrections identified in the gaps analysis.

---

*Document End - Option A vs Parallel System Evaluation*
