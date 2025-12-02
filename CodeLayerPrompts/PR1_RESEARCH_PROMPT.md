# PR1 Codebase Research Prompt

## Command
```
/research_codebase
```

## Prompt

```
Research the Chartsmith codebase (https://github.com/replicatedhq/chartsmith) to answer these questions for our Vercel AI SDK migration:

## Questions to Answer

1. **Current Anthropic SDK Usage**
   - Where is `@anthropic-ai/sdk` imported?
   - How are chat messages currently sent to the API?
   - What's the current streaming implementation?

2. **Chat Component Architecture**
   - Where is the main chat UI component?
   - How is message state currently managed?
   - What hooks or state management patterns are used?

3. **Existing Tools/Function Calling**
   - Are there any tool definitions today?
   - How does Claude receive file context?
   - What chart operations exist as tools?

4. **API Routes**
   - Where are Next.js API routes defined?
   - Is there a current `/api/chat` or similar?
   - How do API routes interact with the Go backend (if at all)?

5. **Test Infrastructure**
   - What test files exist?
   - What testing frameworks are configured?
   - What's the test coverage like?

6. **Environment Variables**
   - What env vars are currently required?
   - Where is `ANTHROPIC_API_KEY` used?
   - What's in `.env.example`?

## Context

We're migrating from `@anthropic-ai/sdk` to Vercel AI SDK with OpenRouter. 

Key changes planned:
- Replace custom chat UI with `useChat` hook from `@ai-sdk/react`
- Replace direct Anthropic calls with `streamText` from `ai` package
- Add provider selection (GPT-4o, Claude) via OpenRouter
- Preserve all existing functionality

## Reference Documents

Read these files in `docs/prds/` for full specifications:
- `PR1_Product_PRD.md` - Functional requirements
- `PR1_Tech_PRD.md` - Technical specification
- `PR1_INSTRUCTIONS.md` - Implementation steps

## Output

Create a research report at `docs/research/PR1_CODEBASE_ANALYSIS.md` with:
- Answers to each question with `file:line` references
- Current architecture diagram (text-based)
- Files that will need modification
- Any gaps or risks identified
```
