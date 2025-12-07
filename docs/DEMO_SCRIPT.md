# Chartsmith AI SDK Migration - Demo Script

**Purpose**: Script for creating demo video showing the AI SDK migration  
**Duration**: ~5-7 minutes  
**Format**: Loom or similar screen recording

---

## Pre-Demo Setup

### 1. Environment Check

```bash
# Terminal 1: Start Go backend
cd chartsmith
make run-worker

# Terminal 2: Start Next.js frontend
cd chartsmith/chartsmith-app
npm run dev

# Terminal 3: Verify services
curl http://localhost:8080/health  # Go backend
curl http://localhost:3000         # Next.js frontend
```

### 2. Test Data

- Ensure database is set up with schema
- Have test workspace ready (optional, can create during demo)

---

## Demo Script

### Segment 1: Introduction & Overview (30 seconds)

**What to Say:**
> "Hi, I'm [Name] and I've completed the Vercel AI SDK migration for Chartsmith. This demo shows how we replaced the custom chat implementation with Vercel AI SDK while maintaining all existing functionality."

**What to Show:**
- Browser open to `http://localhost:3000`
- Application running and responsive
- Quick overview of the UI (chat interface visible)

---

### Segment 2: Application Startup (30 seconds)

**What to Say:**
> "Let me show you the application starting up. The Go backend runs on port 8080, and Next.js runs on port 3000. Both services are running and healthy."

**What to Show:**
1. Terminal showing Go backend logs: `Starting HTTP server for AI SDK tools on port 8080`
2. Terminal showing Next.js: `Ready in X ms`
3. Browser showing application loaded
4. Quick check of browser console (no errors)

**Key Points:**
- ✅ Both services running
- ✅ No console errors
- ✅ Application loads successfully

---

### Segment 3: Chart Creation via Chat (2 minutes)

**What to Say:**
> "Now let's create a Helm chart using natural language. I'll ask Chartsmith to create a simple nginx deployment chart."

**What to Show:**

1. **Type in chat**: "Create a simple nginx deployment chart"
2. **Show streaming response**:
   - Text appears character by character
   - Highlight the smooth streaming
   - Point out the "Thinking..." indicator
   - Show the streaming status

3. **Show AI SDK features**:
   - Tool calls appearing (if visible in UI)
   - File operations happening
   - Real-time updates via Centrifugo

4. **Show result**:
   - Chart files created (Chart.yaml, values.yaml, templates/)
   - File explorer showing new files
   - Code editor showing file contents

**Key Points:**
- ✅ Streaming works smoothly
- ✅ AI understands natural language
- ✅ Files are created correctly
- ✅ Real-time updates work

---

### Segment 4: Plan Workflow (PR3.0) (1 minute)

**What to Say:**
> "One of the key features we implemented is the plan workflow. When the AI proposes changes, users can review and approve them before execution."

**What to Show:**

1. **Send a message that triggers a plan**: "Add a ConfigMap to the nginx chart"
2. **Show PlanChatMessage component**:
   - Plan description displayed
   - Action files listed
   - Proceed/Ignore buttons visible
3. **Click Proceed**:
   - Show execution happening
   - Files being updated
   - Status changes

**Key Points:**
- ✅ Plan workflow working
- ✅ User can review before execution
- ✅ Proceed/Ignore functionality

---

### Segment 5: Tool Calling Demonstration (1 minute)

**What to Say:**
> "The AI can call tools to perform operations. Let me show you how it uses the textEditor tool to modify files."

**What to Show:**

1. **Ask AI to modify a file**: "Update the nginx image tag to 1.25 in values.yaml"
2. **Show tool execution**:
   - AI response mentions using textEditor tool
   - File gets updated
   - Changes visible in editor
3. **Show pending changes**:
   - Yellow dot indicator
   - Commit/Discard buttons

**Key Points:**
- ✅ Tool calling works
- ✅ File operations successful
- ✅ Pending changes tracked

---

### Segment 6: Code Walkthrough - Key Changes (2 minutes)

**What to Say:**
> "Let me walk you through the key code changes. The migration uses an adapter pattern to bridge AI SDK to our existing UI."

**What to Show:**

### 6.1: Adapter Pattern (`hooks/useAISDKChatAdapter.ts`)

**Open File**: `chartsmith-app/hooks/useAISDKChatAdapter.ts`

**Explain:**
- "This adapter bridges the AI SDK `useChat` hook to our existing Message format"
- Show the `useAISDKChatAdapter` function
- Point out message mapping logic
- Show how it integrates with existing ChatContainer

**Key Code Sections:**
```typescript
// Line ~50: useChat hook integration
const { messages, append, ... } = useChat({...})

// Line ~100: Message format conversion
const convertedMessages = mapUIMessagesToMessages(uiMessages)

// Line ~150: Status flag mapping
isThinking: status === 'submitted',
isStreaming: status === 'streaming',
```

---

### 6.2: API Route (`app/api/chat/route.ts`)

**Open File**: `chartsmith-app/app/api/chat/route.ts`

**Explain:**
- "This is the new chat endpoint using Vercel AI SDK's streamText"
- Show the route handler
- Point out tool integration
- Show provider selection

**Key Code Sections:**
```typescript
// Line ~26: AI SDK imports
import { streamText, convertToModelMessages, ... } from 'ai'

// Line ~272: Main streaming call
const result = streamText({
  model: getModel(provider),
  messages: convertedMessages,
  system: systemPrompt,
  tools: createTools(...),
})

// Line ~280: Return stream response
return result.toTextStreamResponse()
```

**Key Points:**
- ✅ Using Vercel AI SDK (`ai` package)
- ✅ No `@anthropic-ai/sdk` imports
- ✅ Multi-provider support via OpenRouter
- ✅ Tool integration working

---

### Segment 7: Feature Flag & Rollback (30 seconds)

**What to Say:**
> "We implemented a feature flag for safe rollout. The AI SDK path is controlled by `NEXT_PUBLIC_USE_AI_SDK_CHAT`."

**What to Show:**

1. **Open `.env.local`** (or show environment variable)
   ```env
   NEXT_PUBLIC_USE_AI_SDK_CHAT=true
   ```

2. **Explain rollback**:
   - "If issues occur, we can instantly rollback by setting this to false"
   - "Both paths use the same database, so no data migration needed"

**Key Points:**
- ✅ Feature flag for safe rollout
- ✅ Instant rollback capability
- ✅ No data migration required

---

### Segment 8: Test Results (30 seconds)

**What to Say:**
> "All tests are passing. We have 116 unit tests and integration tests covering the migration."

**What to Show:**

1. **Run tests**:
   ```bash
   cd chartsmith-app
   npm run test:unit
   ```

2. **Show results**:
   ```
   Test Suites: 9 passed, 9 total
   Tests:       116 passed, 116 total
   ```

**Key Points:**
- ✅ All tests passing
- ✅ Comprehensive test coverage
- ✅ No regressions

---

### Segment 9: Summary & Improvements (30 seconds)

**What to Say:**
> "In summary, we've successfully migrated Chartsmith to Vercel AI SDK. The migration maintains all existing functionality while providing better streaming, multi-provider support, and a more maintainable codebase."

**What to Show:**

- Quick recap of key improvements:
  - ✅ Standardized AI SDK patterns
  - ✅ Multi-provider support (OpenRouter)
  - ✅ Better streaming experience
  - ✅ Simplified state management
  - ✅ All features preserved

---

## Post-Demo Checklist

- [ ] Video shows all required elements:
  - [ ] Application starting successfully
  - [ ] Creating a chart via chat
  - [ ] Streaming responses working
  - [ ] Key code changes walkthrough
- [ ] Video is clear and well-paced (5-7 minutes)
- [ ] Audio quality is good
- [ ] Screen recording captures all details
- [ ] Code walkthrough is easy to follow

---

## Troubleshooting Tips

### If streaming doesn't work:
- Check `NEXT_PUBLIC_USE_AI_SDK_CHAT=true` is set
- Verify API keys are configured
- Check browser console for errors

### If plan workflow doesn't show:
- Ensure PR3.0 features are deployed
- Check that message triggers plan creation
- Verify `buffered_tool_calls` column exists in database

### If tests fail:
- Run `npm install` to ensure dependencies
- Check Node.js version (18+)
- Verify Go version (1.22+)

---

## Alternative Demo Flow (Shorter Version - 3 minutes)

If you need a shorter demo:

1. **Introduction** (15s)
2. **Chart Creation** (1m) - Show streaming and file creation
3. **Code Walkthrough** (1m) - Show adapter and API route
4. **Summary** (15s)

---

## Key Talking Points

### Architecture Decisions
- **Adapter Pattern**: Chose adapter over rewrite (75% less effort)
- **Feature Flag**: Safe rollout with instant rollback
- **Preserved UI**: All existing components kept

### Technical Highlights
- **Zero `@anthropic-ai/sdk`**: Fully migrated to AI SDK Core
- **Multi-Provider**: OpenRouter supports 300+ models
- **Tool Integration**: 6 tools working seamlessly
- **Plan Workflow**: Full parity with legacy path

### Testing
- **116 tests passing**: Comprehensive coverage
- **No regressions**: All existing features work
- **Integration tests**: Tools tested end-to-end

---

**Good luck with your demo! 🎬**

