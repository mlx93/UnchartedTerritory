# PR1 Implementation Agent

## Your Mission
Implement PR1: Vercel AI SDK Foundation for Chartsmith.

---

## Primary Documents (READ FIRST)
1. `PRDs/PR1_Tech_PRD.md` - Your implementation blueprint
2. `PRDs/PR1_Product_PRD.md` - Functional requirements and user stories
3. `PRDs/CHARTSMITH_ARCHITECTURE_DECISIONS.md` - Architectural context

**Note**: `PRDs/` contains planning docs at the workspace root. `chartsmith/` is the forked repo containing the actual codebase. All implementation happens inside `chartsmith/chartsmith-app/`.

## Codebase Reference (Check These)
- `chartsmith/CLAUDE.md` - Codebase entry points and patterns
- `chartsmith/chartsmith-app/CLAUDE.md` - App-specific file locations
- `chartsmith/ARCHITECTURE.md` - Official architecture documentation
- `chartsmith/chartsmith-app/ARCHITECTURE.md` - App-specific architecture
- `ClaudeResearch/VERCEL_AI_SDK_REFERENCE.md` - AI SDK patterns to implement
- `docs/research/PR1_CODEBASE_ANALYSIS.md` - Detailed codebase analysis

---

## Existing Code Context

The current Chartsmith chat uses:
- **Jotai atoms** for state (`atoms/workspace.ts` - contains messagesAtom, plansAtom)
- **Centrifugo WebSocket** for real-time streaming (leave untouched)
- **Go backend** for LLM calls via queue worker (leave untouched in PR1)

Your NEW AI SDK system runs **parallel** to this existing system. Study the existing patterns in `atoms/workspace.ts` and chat components, but don't entangle with them—you're creating an independent path.

---

## What You Are Building

### New Files to Create
| File | Purpose |
|------|---------|
| `chartsmith-app/app/api/chat/route.ts` | NEW API route using streamText |
| `chartsmith-app/lib/ai/provider.ts` | Provider factory for OpenRouter |
| `chartsmith-app/lib/ai/models.ts` | Model definitions |
| `chartsmith-app/lib/ai/config.ts` | AI configuration |
| `chartsmith-app/components/chat/ProviderSelector.tsx` | Model selection UI |

### Files to Modify
- `chartsmith-app/package.json` - Add ai, @ai-sdk/react, @openrouter/ai-sdk-provider
- Chat container component (discover via CLAUDE.md) - Integrate useChat hook
- Message rendering component (discover via CLAUDE.md) - Update for parts-based rendering

### Files to Study (Read Only)
- `chartsmith-app/atoms/workspace.ts` - Understand existing Jotai state patterns (DO NOT MODIFY)

### Files to Delete
- `chartsmith-app/lib/llm/prompt-type.ts` - Orphaned code (never called, verify before deleting)

---

## Critical Constraints

1. **This creates a NEW parallel chat system** - The existing Go-based chat continues to work
2. **Provider selector locks after first message** - Users must start new conversation to switch
3. **No tool implementation in PR1** - Tools come in PR1.5
4. **No Go changes in PR1** - Go backend unchanged

---

## Environment Variables to Add
```env
OPENROUTER_API_KEY=sk-or-v1-xxxxx
DEFAULT_AI_PROVIDER=openai
DEFAULT_AI_MODEL=openai/gpt-4o
```

---

## Success Criteria (All Must Pass)
- [ ] POST /api/chat returns streaming response
- [ ] useChat hook integrates with new route
- [ ] ProviderSelector shows GPT-4o and Claude options
- [ ] Provider selection locks after first message
- [ ] Streaming displays correctly in UI
- [ ] No TypeScript errors
- [ ] Existing tests pass or are updated

---

## Implementation Order

0. **Discover existing chat architecture** - Read `chartsmith/CLAUDE.md` and `chartsmith/chartsmith-app/CLAUDE.md`. Find the ChatContainer component, understand Jotai atom patterns in `atoms/workspace.ts`, identify where messages are rendered.
1. **Install dependencies** - ai, @ai-sdk/react, @openrouter/ai-sdk-provider
2. **Create provider factory** - `lib/ai/provider.ts` with getModel() function
3. **Create API route** - `app/api/chat/route.ts` with streamText
4. **Create ProviderSelector** - UI component for model selection
5. **Integrate useChat** - Wire up to chat components discovered in step 0
6. **Test streaming** - Verify end-to-end flow works
7. **Clean up** - Verify `lib/llm/prompt-type.ts` is orphaned, then delete

---

## COMPLETION REQUIREMENTS

### When you have completed ALL success criteria, create the following documentation:

**Create file: `docs/PR1_COMPLETION_REPORT.md`**

This file MUST contain:

### 1. Files Summary Table
```markdown
| Action | File Path | Description |
|--------|-----------|-------------|
| Created | ... | ... |
| Modified | ... | ... |
| Deleted | ... | ... |
```

### 2. Implementation Notes
- Any deviations from the spec with rationale
- Challenges encountered and how they were resolved
- Patterns discovered that may be useful for PR1.5

### 3. Provider Factory Exports
Document the exact exports from `lib/ai/provider.ts`:
- Function signatures for getModel()
- AVAILABLE_PROVIDERS constant structure

### 4. API Route Contract
Document the exact request/response format for `/api/chat`:
```typescript
// Request body structure
// Response format
```

### 5. Test Results
- Screenshot or text output showing tests pass
- Manual testing results for streaming

### 6. Known Issues / TODOs for PR1.5
- List any incomplete items or known limitations
- Dependencies PR1.5 needs to be aware of

### 7. PR1.5 Readiness Checklist
Confirm these are ready:
- [ ] /api/chat/route.ts exists and streams correctly
- [ ] Provider factory exports getModel()
- [ ] Provider factory exports AVAILABLE_PROVIDERS
- [ ] ProviderSelector UI functional
- [ ] useChat hook integrated
- [ ] No console errors

---

## DO NOT

- Implement tools (that's PR1.5)
- Modify Go code (Go changes are PR1.5)
- Add workspace context (that's PR1.5)
- Create validation logic (that's PR2)
- Start on the next PR without creating the completion report

---

*When your completion report is ready, the Master Orchestrator will review it and provide you with the PR1.5 sub-agent prompt.*

