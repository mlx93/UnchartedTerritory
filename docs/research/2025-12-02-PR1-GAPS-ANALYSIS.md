# PR1 PRD Gaps Analysis: Chartsmith Vercel AI SDK Migration

---
date: 2025-12-02T04:02:42Z
researcher: Claude Code
git_commit: e615830f7b0b60ecb817d762b9b4e4186f855d85
branch: main
repository: UnchartedTerritory/chartsmith
topic: "PR1 PRD Gaps Analysis Against Actual Chartsmith Codebase"
tags: [research, gaps-analysis, PR1, chartsmith, vercel-ai-sdk]
status: complete
last_updated: 2025-12-02
last_updated_by: Claude Code
---

## Executive Summary

After thorough codebase analysis, this document identifies **gaps, inaccuracies, and missing information** in the PR1 PRD documents (PR1_Product_PRD.md and PR1_Tech_PRD.md) and ClaudeResearch files when compared against the actual Chartsmith codebase.

### Critical Findings Summary

| Category | Finding | Severity |
|----------|---------|----------|
| **Orphaned Code** | `promptType()` function is unused anywhere in codebase | HIGH |
| **Model Names** | PRDs reference non-existent "claude-4.5-sonnet" model | MEDIUM |
| **New Chart Flow** | Special revision-0 workflow not documented in PRDs | MEDIUM |
| **Architecture Doc** | Some claims need updating based on actual patterns | LOW |
| **Test Strategy** | Missing specific test file references | LOW |

---

## 1. Critical Gap: Frontend Anthropic SDK Usage is Orphaned

### What PRD Says

**PR1_Tech_PRD.md (Line 126)**:
> | `@anthropic-ai/sdk` | Replaced by AI SDK + OpenRouter |

**CURRENT_STATE_ANALYSIS.md (Lines 63-67)**:
> **Single Anthropic SDK Usage**:
> - **File**: `chartsmith-app/lib/llm/prompt-type.ts:1`
> - **Purpose**: Classify user prompts as "plan" or "chat"

### What Codebase Analysis Reveals

**The `promptType()` function is NOT USED anywhere in the codebase.**

Comprehensive search across all TypeScript/JavaScript files found:
- **No imports** of `promptType`, `PromptType`, `PromptRole`, or `PromptIntent` from `lib/llm/prompt-type.ts`
- **No consumers** of this functionality in any component, hook, or API route
- The function exists but appears to be **orphaned/dead code**

### Impact on PR1

| Option | Description | Recommendation |
|--------|-------------|----------------|
| **Option A** | Remove `@anthropic-ai/sdk` and delete `prompt-type.ts` entirely | Simplest - dead code removal |
| **Option B** | Integrate `promptType()` with new AI SDK chat flow | Only if intent classification is desired |
| **Option C** | Keep existing code, add parallel AI SDK system | Bloated, not recommended |

**Recommendation**: Verify with stakeholder whether `promptType()` functionality is needed. If not, PR1 simplifies to:
- Adding new `/api/chat` route with AI SDK
- Adding new chat components with `useChat`
- **NOT needing to "migrate" anything** since frontend SDK is unused

### Files Affected
- `chartsmith-app/lib/llm/prompt-type.ts` - Orphaned, can be deleted
- `chartsmith-app/package.json:21` - `@anthropic-ai/sdk` can be removed if no usage

---

## 2. Model Name Discrepancies

### What PRDs Say

**PR1_Tech_PRD.md (Lines 221-223)**:
```
"anthropic" -> openrouter("anthropic/claude-4.5-sonnet") OR custom modelId
```

**PR1_Product_PRD.md (Lines 152-156)**:
```
| Anthropic | `anthropic/claude-4.5-sonnet` | Claude 4.5 Sonnet |
```

### What Actually Exists

**Anthropic models as of Jan 2025**:
- `claude-3-5-sonnet-20241022` (Claude 3.5 Sonnet)
- `claude-3-7-sonnet-20250219` (Claude 3.7 Sonnet) - Used in Go backend
- **There is NO "claude-4.5-sonnet"**

**OpenRouter model IDs**:
- `anthropic/claude-3.5-sonnet` (latest)
- `anthropic/claude-3-5-sonnet-20241022` (dated version)

### Corrections Needed

| PRD Location | Current | Correct |
|--------------|---------|---------|
| PR1_Tech_PRD.md:222 | `anthropic/claude-4.5-sonnet` | `anthropic/claude-3.5-sonnet` |
| PR1_Product_PRD.md:154 | `anthropic/claude-4.5-sonnet` | `anthropic/claude-3.5-sonnet` |
| ARCHITECTURE_DECISIONS.md:143 | `anthropic/claude-3.5-sonnet` | Already correct |

---

## 3. New Chart Creation Flow Not Documented

### What Codebase Shows

**ChatContainer.tsx (Lines 75-94)** implements special handling for new charts:

```typescript
if (workspace?.currentRevisionNumber === 0) {
  return <NewChartContent ... />
}
```

This flow:
- Shows `<NewChartContent>` instead of main chat UI
- Uses simplified submit handler with hardcoded "auto" role
- No role selector displayed
- Different component: `<NewChartChatMessage>` instead of `<ChatMessage>`

### What PRDs Miss

Neither PR1_Product_PRD.md nor PR1_Tech_PRD.md mention:
1. Special handling for `revisionNumber === 0`
2. Different UI components for new chart creation
3. Role selector visibility rules

### Recommended PRD Addition

```markdown
### FR-1.4: New Chart Creation Mode

When `workspace.currentRevisionNumber === 0`:
- Render `NewChartContent` component instead of main chat
- Hide role selector (use "auto" role)
- Display "Create a new Helm chart" header
- Show "Create Chart" button when plan status is "review"
```

---

## 4. ClaudeResearch File Accuracy Assessment

### CURRENT_STATE_ANALYSIS.md

| Section | Status | Notes |
|---------|--------|-------|
| System Architecture diagram | ACCURATE | Matches actual flow |
| Frontend Anthropic SDK | INACCURATE | Claims it's "used", but it's orphaned |
| Go Backend SDK files | ACCURATE | 10 files confirmed |
| Models Used | ACCURATE | claude-3-7-sonnet, claude-3-5-sonnet confirmed |
| Directory Structure | ACCURATE | Paths verified |
| Chat Message Flow | ACCURATE | Flow verified |
| Tool Calling | ACCURATE | 3 tools confirmed |
| Environment Variables | ACCURATE | All variables verified |
| Migration Opportunities | NEEDS UPDATE | Should note promptType is unused |

### ARCHITECTURE_DECISIONS.md

| Section | Status | Notes |
|---------|--------|-------|
| ADR-001 to ADR-007 | ACCURATE | Decisions align with requirements |
| Target Architecture | ACCURATE | Parallel system approach confirmed |
| API Contracts | NEEDS UPDATE | Model names incorrect |

### VERCEL_AI_SDK_REFERENCE.md

| Section | Status | Notes |
|---------|--------|-------|
| Package versions | ACCURATE | Current versions listed |
| useChat patterns | ACCURATE | Correct implementation patterns |
| Provider configuration | ACCURATE | OpenRouter config correct |
| Tool calling patterns | ACCURATE | Matches AI SDK docs |
| Compatibility Notes | ACCURATE | Two-system coexistence noted |

### VALIDATION_TOOLS_REFERENCE.md

| Section | Status | Notes |
|---------|--------|-------|
| Validation Stack | ACCURATE | helm lint/template/kube-score |
| Tool definitions | ACCURATE | Matches PR2 requirements |
| AI Integration Points | ACCURATE | Tool schema correct |

---

## 5. Test Strategy Gaps

### What PRD Says

**PR1_Tech_PRD.md (Lines 489-529)** mentions tests generically:
- "Provider factory tests"
- "Component tests"
- "API route tests"

### What's Missing

Specific test file references that need updating:

**Jest Unit Tests to Update/Create**:
| File | Current Tests | PR1 Changes Needed |
|------|---------------|-------------------|
| `atoms/__tests__/workspace.test.ts` | Render deduplication | May need message state tests |
| `components/__tests__/FileTree.test.ts` | Patch stats | No changes |
| `hooks/__tests__/parseDiff.test.ts` | Diff parsing | No changes |
| **NEW** `lib/ai/__tests__/provider.test.ts` | N/A | Create for provider factory |
| **NEW** `components/chat/__tests__/ProviderSelector.test.ts` | N/A | Create for selector |

**Playwright E2E Tests to Consider**:
| File | Current Tests | PR1 Impact |
|------|---------------|------------|
| `tests/chat-scrolling.spec.ts` | Auto-scroll | May need update for new chat components |
| `tests/upload-chart.spec.ts` | Chart upload | Workflow unchanged |
| **NEW** `tests/provider-switching.spec.ts` | N/A | Create for provider selection |

---

## 6. Component Reference Corrections

### ChatContainer.tsx

**CURRENT_STATE_ANALYSIS.md states**: `components/ChatContainer.tsx:18`

**Actual**:
- Main component export at line 18 ✓
- Form submission handler at lines 48-58
- New chart mode check at line 75
- Role selector at lines 112-140

### ChatMessage.tsx

**CURRENT_STATE_ANALYSIS.md states**: `components/ChatMessage.tsx:72`

**Actual**:
- Main component at line 72 ✓
- Message parts rendering at lines 149-219 (SortedContent)
- Loading states at lines 238-261
- Followup actions at lines 309-330

---

## 7. API Route Clarifications

### What Exists (Verified)

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/auth/status` | GET | Token validation |
| `/api/config` | GET | Public env config |
| `/api/push` | GET | Centrifugo token |
| `/api/upload-chart` | POST | Chart upload |
| `/api/workspace/[id]` | GET | Get workspace |
| `/api/workspace/[id]/message` | POST | Create message |
| `/api/workspace/[id]/messages` | GET | List messages |
| `/api/workspace/[id]/plans` | GET | List plans |
| `/api/workspace/[id]/renders` | GET | List renders |
| `/api/workspace/[id]/revision` | POST | Create revision |

### What PRD Should Clarify

**PR1 creates NEW route**: `/api/chat` (does NOT exist currently)

This is a **parallel system** - existing routes remain unchanged.

The PRD should explicitly state:
```markdown
### API Route Strategy

**NEW (PR1 Creates)**:
- `POST /api/chat` - Vercel AI SDK streamText endpoint

**UNCHANGED (Existing)**:
- All `/api/workspace/*` routes continue to work
- Go backend continues processing via work queue
- Centrifugo real-time updates unchanged
```

---

## 8. State Management Clarifications

### Jotai Atoms (Verified)

| Atom | Line | Type |
|------|------|------|
| `workspaceAtom` | 6 | Workspace object |
| `messagesAtom` | 10 | Message[] |
| `plansAtom` | 16 | Plan[] |
| `rendersAtom` | 22 | RenderedWorkspace[] |
| `conversionsAtom` | 34 | Conversion[] |
| `isRenderingAtom` | 259-261 | Computed boolean |
| `activeRenderIdsAtom` | 256 | string[] |

### PR1 Impact on State

**What PRD says**: Replace custom state with `useChat` hook

**What should happen**:
- NEW chat components use `useChat` managed state
- EXISTING Jotai atoms remain for workspace functionality
- Two state systems coexist (documented in ARCHITECTURE_DECISIONS.md)

**Clarification needed**: PRD should explicitly state which atoms are affected:
- `messagesAtom` - May be partially replaced for new chat flow
- Other atoms - Unchanged

---

## 9. Streaming Implementation Details

### What PRD Says

**PR1_Tech_PRD.md (Lines 447-482)**: Generic useChat configuration

### What Should Be Added

Chartsmith-specific streaming considerations:

```markdown
### Streaming Coexistence

PR1 introduces a SECOND streaming system:

| System | Protocol | Usage |
|--------|----------|-------|
| **Existing** | Centrifugo WebSocket | Workspace updates, plan progress |
| **NEW (PR1)** | AI SDK Data Stream | New /api/chat responses |

Both systems will run in parallel. The existing Centrifugo system
continues to handle:
- Plan updates (`plan-updated` events)
- Render streaming (`render-stream` events)
- Artifact updates (`artifact-updated` events)
- Conversion progress (`conversion-status` events)
```

---

## 10. Environment Variable Clarifications

### Current Variables (Verified)

| Variable | Used By | Required For |
|----------|---------|--------------|
| `ANTHROPIC_API_KEY` | Go backend (pkg/param/param.go:17) | LLM calls |
| `ANTHROPIC_API_KEY` | TS (lib/llm/prompt-type.ts:22) | UNUSED function |
| `GROQ_API_KEY` | Go backend | Intent classification |
| `VOYAGE_API_KEY` | Go backend | Embeddings |
| `CHARTSMITH_PG_URI` | Both | Database |
| `CENTRIFUGO_*` | Both | Real-time |

### PR1 New Variables

| Variable | Purpose | Required |
|----------|---------|----------|
| `OPENROUTER_API_KEY` | OpenRouter API access | YES |
| `OPENAI_API_KEY` | Direct fallback | NO |
| `DEFAULT_AI_PROVIDER` | Default provider | NO |
| `DEFAULT_AI_MODEL` | Default model | NO |

### Clarification

**`ANTHROPIC_API_KEY` for TypeScript**:
- Currently required by `prompt-type.ts` but function is unused
- If `prompt-type.ts` is deleted, this variable is NOT needed in frontend
- Go backend still requires it

---

## Summary of Required PRD Updates

### PR1_Product_PRD.md

1. Add FR-1.4 for New Chart Creation Mode
2. Fix model name: `anthropic/claude-3.5-sonnet`
3. Clarify provider selector visibility rules

### PR1_Tech_PRD.md

1. Fix model names throughout
2. Clarify that `@anthropic-ai/sdk` removal may be simple deletion (not migration)
3. Add specific test file references
4. Document streaming coexistence
5. Add environment variable clarification for unused `ANTHROPIC_API_KEY`

### CURRENT_STATE_ANALYSIS.md

1. Update "Frontend Anthropic SDK Usage" section to note it's orphaned
2. Add note about `promptType()` being unused

### ARCHITECTURE_DECISIONS.md

1. Fix model name in ADR-002 examples

---

## Recommendations Before PR1 Implementation

1. **Confirm with stakeholder**: Is `promptType()` functionality needed?
   - If NO: Delete `lib/llm/prompt-type.ts` and remove `@anthropic-ai/sdk` from package.json
   - If YES: Document how it should integrate with new chat flow

2. **Update model names**: Use correct OpenRouter model IDs

3. **Document New Chart Mode**: Add handling specification to PRD

4. **Create test plan**: List specific test files to create/update

5. **Verify environment strategy**: Confirm frontend doesn't need `ANTHROPIC_API_KEY`

---

## Code References

| File | Line | Description |
|------|------|-------------|
| `lib/llm/prompt-type.ts` | 1 | Orphaned Anthropic SDK import |
| `components/ChatContainer.tsx` | 75 | New chart mode check |
| `components/ChatContainer.tsx` | 48-58 | Form submission handler |
| `atoms/workspace.ts` | 10 | messagesAtom definition |
| `hooks/useCentrifugo.ts` | 492-607 | WebSocket connection setup |
| `lib/utils/queue.ts` | 9-20 | Work queue enqueue function |
| `pkg/llm/execute-action.go` | 510-532 | text_editor tool definition |
| `pkg/llm/conversational.go` | 99-128 | Conversational tools |

---

*Document End - PR1 Gaps Analysis*
