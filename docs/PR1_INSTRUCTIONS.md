# PR1 Implementation Instructions

## Pre-Implementation: Repo Exploration (30-60 min)

### Step 1: Fork & Clone
```bash
git clone https://github.com/replicatedhq/chartsmith
cd chartsmith
```

### Step 2: Map Current Chat Implementation
Find and document actual file paths for:

```bash
# Find Anthropic SDK usage
grep -r "@anthropic-ai/sdk" --include="*.ts" --include="*.tsx"
grep -r "anthropic" --include="*.ts" --include="*.tsx"

# Find chat components
find . -name "*[Cc]hat*" -type f

# Find API routes
find . -path "*/api/*" -name "*.ts"

# Find existing hooks
find . -name "use*.ts" -o -name "use*.tsx"
```

### Step 3: Document Current State
Update `CURRENT_STATE_ANALYSIS.md` with actual paths:
- [ ] Chat component location(s)
- [ ] Anthropic SDK import locations
- [ ] Current API route handlers
- [ ] Custom streaming implementation files
- [ ] State management hooks/files

### Step 4: Inventory Existing Tests
```bash
# Find test files
find . -name "*.test.ts" -o -name "*.test.tsx" -o -name "*_test.go"

# Check test directories
ls -la test_chart/ testdata/
```

Document what tests exist and what they cover.

### Step 5: Verify Dev Environment
```bash
# Frontend
cd chartsmith-app
npm install
npm run dev

# Confirm app runs before making changes
```

---

## Implementation Sequence (Days 1-3)

### Day 1: Foundation
1. Create feature branch: `git checkout -b feat/vercel-ai-sdk-migration`
2. Install dependencies:
   ```bash
   npm install ai @ai-sdk/react @openrouter/ai-sdk-provider zod
   npm uninstall @anthropic-ai/sdk  # After migration complete
   ```
3. Add env vars to `.env.example`:
   ```
   OPENROUTER_API_KEY=sk-or-v1-xxxxx
   DEFAULT_AI_PROVIDER=openai
   DEFAULT_AI_MODEL=openai/gpt-4o
   ```
4. Create provider factory: `src/lib/ai/provider.ts`
5. Create new API route: `src/app/api/chat/route.ts`
6. Test basic streaming works

### Day 2: Component Migration
1. Create `ProviderSelector.tsx` component
2. Migrate main Chat component to `useChat` hook
3. Update message rendering for parts-based format
4. Migrate/preserve existing tools
5. Test file context and chart operations

### Day 3: Polish & Testing
1. Fix/update failing tests
2. Add new unit tests for provider factory
3. Run full test suite
4. Update `ARCHITECTURE.md`
5. Final manual QA
6. Clean commit history, open PR

---

## Key Files to Create

| File | Purpose |
|------|---------|
| `src/lib/ai/provider.ts` | Provider factory (OpenRouter) |
| `src/lib/ai/models.ts` | Model definitions |
| `src/lib/ai/config.ts` | AI configuration |
| `src/app/api/chat/route.ts` | New chat API route |
| `src/components/chat/ProviderSelector.tsx` | Model selector UI |

## Key Files to Modify
- Main chat component (path TBD from exploration)
- Message rendering component (path TBD)
- `package.json`
- `chartsmith-app/ARCHITECTURE.md`

## Validation Checklist
- [ ] Chat works identically to before
- [ ] Streaming has no visible degradation
- [ ] Provider selector shows GPT-4o and Claude options
- [ ] All existing tests pass
- [ ] VS Code extension auth still works (`/api/auth/status`)
- [ ] No console errors
