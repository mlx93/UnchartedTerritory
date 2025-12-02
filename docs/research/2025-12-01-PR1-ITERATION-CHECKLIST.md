# PR1 PRD Iteration Checklist

This document lists all items that need to be addressed when iterating on the existing PR1 PRDs.

**Overall Assessment**: The current PR1 PRDs are **~90% correct** and follow the proper parallel system approach. Only targeted corrections are needed - NOT a full rewrite.

---

## Critical Fixes (Must Address)

### 1. Model Name Typo

**Issue**: References non-existent "claude-4.5-sonnet" model

| File | Line | Current | Correct |
|------|------|---------|---------|
| `PR1_Product_PRD.md` | 155 | `anthropic/claude-4.5-sonnet` | `anthropic/claude-3.5-sonnet` |
| `PR1_Product_PRD.md` | 262 | `Claude 4.5 Sonnet` | `Claude 3.5 Sonnet` |
| `PR1_Product_PRD.md` | 347 | `anthropic/claude-4.5-sonnet` | `anthropic/claude-3.5-sonnet` |
| `PR1_Tech_PRD.md` | 221 | `anthropic/claude-4.5-sonnet` | `anthropic/claude-3.5-sonnet` |
| `PR1_Tech_PRD.md` | 740 | `anthropic/claude-4.5-sonnet` | `anthropic/claude-3.5-sonnet` |

**Context**: Claude 4.5 does not exist. Current models are:
- `claude-3-5-sonnet-20241022` (Claude 3.5 Sonnet)
- `claude-3-7-sonnet-20250219` (Claude 3.7 Sonnet - used in Go backend)

---

### 2. Dead Code Cleanup Not Specified

**Issue**: PRDs don't explicitly mention deleting orphaned code

**Add to PR1_Tech_PRD.md Section "Files to Remove/Deprecate" (after line 197)**:

```markdown
### Files to Delete (Orphaned Code)

| File | Reason |
|------|--------|
| `lib/llm/prompt-type.ts` | Orphaned - function not imported or used anywhere in codebase |

**Note**: The `promptType()` function in this file was intended for prompt classification but is never called. The actual intent classification happens in the Go backend via the `new_intent` queue handler.
```

**Add to PR1_Tech_PRD.md "Removed Dependencies" section (line 122-125)**:

```markdown
### Removed Dependencies
| Package | Reason |
|---------|--------|
| `@anthropic-ai/sdk` | Replaced by AI SDK + OpenRouter. Frontend usage was orphaned code. |
```

---

### 3. New Chart Mode Not Documented

**Issue**: `ChatContainer.tsx:75-94` has special handling for `revisionNumber === 0` that isn't documented

**Add to PR1_Product_PRD.md after FR-1.3 (around line 147)**:

```markdown
#### FR-1.4: New Chart Creation Mode

When `workspace.currentRevisionNumber === 0`:
- Render `NewChartContent` component instead of main chat interface
- Hide role selector (use "auto" role by default)
- Display "Create a new Helm chart" header
- Show "Create Chart" button when plan status is "review"
- Simplified submit handler without role selection

**Existing Behavior**: This mode is already implemented in the existing codebase. The new AI SDK chat should preserve this behavior when integrated.
```

---

## Medium Priority Fixes

### 4. Clarify Tool Strategy for PR1

**Issue**: PRDs are ambiguous about whether the new AI SDK chat will have tool support

**Current text** (PR1_Tech_PRD.md lines 385-411) mentions "Existing Tool Migration" but this is confusing because:
- The existing Go tools stay in Go (unchanged)
- The new AI SDK chat needs its own tool definitions
- PR1 scope may not include tools for the new chat

**Clarify in PR1_Tech_PRD.md** by replacing "Existing Tool Migration" section:

```markdown
### Tool Strategy for PR1

**PR1 Scope**: The NEW `/api/chat` route will be **conversational only** (no tools).

**Rationale**:
- Existing Go tools (`text_editor`, `subchart_version`, `kubernetes_version`) remain in Go backend
- These tools require file system access and complex logic that should stay in Go
- PR2 adds the validation tool which will establish the Go HTTP API pattern
- Full tool migration is a future enhancement, not PR1 scope

**What This Means**:
- Users can use the new AI SDK chat for conversational questions
- File editing and chart operations continue to use the existing Go-based flow
- This is acceptable because both systems coexist

**Future Enhancement** (Post-PR2):
If tool support is needed in the new chat, options include:
1. Add HTTP endpoints to Go for tool execution (established in PR2)
2. Call Go endpoints from AI SDK tool `execute()` functions
3. Mirror simple tools in TypeScript (e.g., kubernetes_version)
```

---

### 5. File Path Corrections

**Issue**: Some file paths use `src/` prefix which may not match actual chartsmith structure

**Verify and correct paths in PR1_Tech_PRD.md**:

| Current Path | Actual Path (verify) |
|--------------|---------------------|
| `src/lib/ai/provider.ts` | `lib/ai/provider.ts` |
| `src/components/chat/ProviderSelector.tsx` | `components/chat/ProviderSelector.tsx` |
| `src/app/api/chat/route.ts` | `app/api/chat/route.ts` |

**Note**: The chartsmith-app structure doesn't use a `src/` directory. Paths should be relative to `chartsmith-app/`.

---

### 6. Environment Variable Clarification

**Issue**: Frontend `ANTHROPIC_API_KEY` situation not explained

**Add note to PR1_Tech_PRD.md "Environment Configuration" section**:

```markdown
### Environment Variable Notes

**Regarding `ANTHROPIC_API_KEY`**:
- Currently in `.env` for both frontend and Go backend
- Frontend usage (`lib/llm/prompt-type.ts`) is orphaned code being deleted
- After PR1, frontend no longer needs `ANTHROPIC_API_KEY`
- Go backend continues to use `ANTHROPIC_API_KEY` (unchanged)

**New Variables (Frontend Only)**:
- `OPENROUTER_API_KEY` - Required for new AI SDK chat
- `OPENAI_API_KEY` - Optional fallback
```

---

## Low Priority / Nice-to-Have

### 7. Add Reference to Research Documents

**Add to both PRDs in a new "Related Documents" section**:

```markdown
## Related Documents

| Document | Purpose |
|----------|---------|
| `docs/research/2025-12-01-OPTION-A-VS-PARALLEL-EVALUATION.md` | Architecture decision rationale |
| `docs/research/2025-12-02-COMPREHENSIVE-PR1-RESEARCH.md` | Full codebase verification |
| `docs/research/2025-12-02-PR1-GAPS-ANALYSIS.md` | PRD gaps analysis |
| `ClaudeResearch/CURRENT_STATE_ANALYSIS.md` | Current architecture details |
| `ClaudeResearch/VERCEL_AI_SDK_REFERENCE.md` | SDK implementation patterns |
```

---

### 8. Test File Specificity

**Issue**: Test strategy is generic, doesn't reference actual test files

**Add to PR1_Tech_PRD.md "Testing Strategy" section**:

```markdown
### Existing Test Files to Update

| File | Changes Needed |
|------|----------------|
| `atoms/__tests__/workspace.test.ts` | May need message state tests for new flow |
| `components/__tests__/FileTree.test.ts` | No changes expected |
| `hooks/__tests__/parseDiff.test.ts` | No changes expected |

### New Test Files to Create

| File | Purpose |
|------|---------|
| `lib/ai/__tests__/provider.test.ts` | Provider factory unit tests |
| `components/chat/__tests__/ProviderSelector.test.ts` | Component tests |
| `app/api/chat/__tests__/route.test.ts` | API route integration tests |
```

---

### 9. Streaming Coexistence Documentation

**Issue**: Not clear that two streaming systems will coexist

**Add to PR1_Tech_PRD.md "Streaming Implementation" section**:

```markdown
### Streaming System Coexistence

PR1 introduces a **second streaming system** alongside the existing one:

| System | Protocol | Used By |
|--------|----------|---------|
| **Existing** | Centrifugo WebSocket | Workspace operations, plan progress, file updates |
| **NEW (PR1)** | AI SDK Data Stream (SSE) | New `/api/chat` responses |

**Both systems run in parallel**. The existing Centrifugo system continues to handle:
- `plan-updated` events
- `chatmessage-updated` events (for existing flow)
- `render-stream` events
- `artifact-updated` events
- `conversion-status` events

The new AI SDK Data Stream handles:
- Responses from the new `/api/chat` endpoint only
```

---

## Summary Checklist

### Must Fix Before Implementation

- [ ] Fix model name: `claude-4.5-sonnet` → `claude-3.5-sonnet` (5 locations)
- [ ] Add: Delete `lib/llm/prompt-type.ts` to "Files to Remove"
- [ ] Add: FR-1.4 New Chart Creation Mode
- [ ] Clarify: Tool strategy (conversational only for PR1)

### Should Fix

- [ ] Correct file paths (remove `src/` prefix)
- [ ] Add environment variable clarification
- [ ] Add related documents section

### Nice to Have

- [ ] Add specific test file references
- [ ] Add streaming coexistence documentation

---

## Notes for Plan Iteration

1. **Do NOT rewrite the PRDs** - they are fundamentally correct
2. **Use targeted edits** - fix specific lines/sections
3. **Preserve the parallel system approach** - this is correct per requirements
4. **Keep Go backend unchanged** - this is explicitly required
5. **Reference this checklist** - use as verification after iteration

---

*Document End - PR1 Iteration Checklist*
