# Chartsmith AI SDK Migration - Demo Script (PR4.0)

**Purpose**: Script for creating demo video showcasing AI SDK migration with PR4.0 features  
**Duration**: ~5 minutes  
**Format**: Loom or similar screen recording

---

## Architecture Overview (What to Say)

> "Before diving into the demo, let me briefly explain the architecture changes. Chartsmith originally used a custom Go backend that made direct Anthropic SDK calls—all LLM communication went through PostgreSQL notifications to a Go worker. This worked, but it locked us into a single provider and made the chat system difficult to extend.
>
> For this migration, I used an **adapter pattern**. Rather than rewriting the entire UI—which would have taken over a hundred hours—I created a thin adapter layer that bridges Vercel AI SDK's `useChat` hook to our existing React components. The adapter converts between AI SDK's message format and our internal format, so all 27+ existing UI features just work.
>
> The key change is where LLM calls happen. Previously, the Go backend orchestrated everything. Now, TypeScript handles all AI interactions through a new `/api/chat` streaming endpoint using AI SDK's `streamText`. Go still runs, but only for deterministic operations like file system access and Helm commands—it no longer makes any LLM calls.
>
> This also unlocked multi-provider support. Instead of being locked to Anthropic, we can now switch between Claude, GPT-4, and other models at runtime through OpenRouter. You'll see that in action later in the demo.
>
> The best part? All the existing UI components, our Jotai state management, the Centrifugo WebSocket updates, and database persistence—none of that changed. We swapped the transport layer without touching the UI."

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

### 2. Pre-Demo Checklist
- [ ] Fresh workspace (no existing chart files)
- [ ] Multiple API keys configured (Claude + OpenAI)
- [ ] Browser developer tools closed (cleaner recording)

---

## Demo Script (5 Minutes)

### Segment 1: Introduction (30 seconds)

**What to Say:**
> "Hi, I'm [Name]. For this challenge, I forked Replicated's Chartsmith—a production-grade Helm chart development platform with a complex Go and TypeScript architecture. It was the perfect candidate for a Vercel AI SDK migration because it had a custom Anthropic SDK implementation locked to a single provider.
>
> My goal was to replace that custom chat system with Vercel AI SDK while preserving all existing functionality. Let me show you the result."

**What to Show:**
- Browser open to `http://localhost:3000`
- Clean workspace visible
- Chat interface ready

---

### Segment 2: Application Startup (15 seconds)

**What to Say:**
> "The Go backend runs on port 8080, Next.js on 3000. Both services are healthy and ready."

**What to Show:**
1. Terminal showing Go backend: `Starting HTTP server for AI SDK tools on port 8080`
2. Terminal showing Next.js: `Ready in X ms`
3. Browser showing application loaded

**Key Points:**
- ✅ Both services running
- ✅ Application loads successfully

---

### Segment 3: Chart Creation + State Persistence (1 minute 30 seconds)

**What to Say:**
> "Let's create a Helm chart using natural language. I'll ask Chartsmith to create a simple nginx deployment."

**What to Show:**

#### 3.1: Create the Chart
1. **Type in chat**: `"Create a simple nginx deployment"`
2. **Show streaming response**:
   - Text appears character by character
   - Show the "Thinking..." indicator transitioning to streaming
   - Tool calls happening (file operations)

3. **Show result**:
   - Chart files created (Chart.yaml, values.yaml, templates/)
   - File explorer showing new files
   - Code editor showing file contents

4. **Show Proceed/Ignore buttons** (PR4.0 fix):
   - Point out: "Notice the Proceed and Ignore buttons appearing—this was fixed in PR4.0"
   - Buttons appear after ~500ms once the plan is linked

#### 3.2: State Persistence (Page Refresh)

**What to Say:**
> "Let me refresh the page to demonstrate state persistence. All our conversation history and files remain intact."

**What to Show:**
1. **Press F5 or click refresh**
2. **Show page reloading**
3. **Point out**:
   - Conversation history still visible
   - Chart files still in file explorer
   - Workspace state fully preserved

**Key Points:**
- ✅ Streaming works smoothly
- ✅ Proceed/Ignore buttons appear correctly (PR4.0 fix)
- ✅ Full state persistence across page refresh
- ✅ Files created correctly

---

### Segment 4: Chart Validation (1 minute) ← NEW in PR4.0

**What to Say:**
> "Now let's validate our chart. Chartsmith runs a full validation pipeline—Helm lint, Helm template, and kube-score—then the AI interprets the results."

**What to Show:**

1. **Type in chat**: `"Validate my chart"`

2. **Show validation in progress**:
   - AI calling the validateChart tool
   - Processing indicator

3. **Show validation results component**:
   - Overall status badge (pass/warning/fail)
   - **Helm Lint**: Results with any warnings
   - **Helm Template**: Success with resource count
   - **Kube-score**: Findings with severity colors:
     - 🔴 Critical (red)
     - 🟡 Warning (yellow)
     - 🔵 Info (blue)

4. **Show AI interpretation**:
   - Natural language explanation of issues
   - Specific fix suggestions for each finding
   - Point out: "The AI identifies that we're missing resource limits—a common best practice issue"

**Key Points:**
- ✅ Validation pipeline runs end-to-end
- ✅ Results displayed with severity indicators
- ✅ AI provides actionable fix suggestions

---

### Segment 5: Live Provider Switching + Fixing Issues (45 seconds) ← NEW in PR4.0

**What to Say:**
> "Let's fix the resource limits issue. Then I'll switch to a different AI provider mid-conversation—notice how the history is preserved."

**What to Show:**

#### 5.1: Add Resource Limits (with Claude)
1. **Type in chat**: `"Add resource limits to the deployment"`
2. **Show response**:
   - AI proposes changes to deployment template
   - Resource limits added (requests/limits for CPU and memory)
3. **Click Proceed** to apply changes

#### 5.2: Switch Provider
1. **Click provider dropdown** in chat header (LiveProviderSwitcher)
2. **Select GPT-4o** (or another available provider)
3. **Point out**: "Watch—I'm switching from Claude to GPT-4o"

#### 5.3: Continue with New Provider
1. **Type in chat**: `"Lower the resource limits"`
2. **Show response from GPT-4o**:
   - AI modifies the resource values
   - Different prose style (subtle difference)
3. **Point out**: "The conversation history is fully preserved, and GPT-4o picks up right where Claude left off"

**Key Points:**
- ✅ Provider switching works mid-conversation
- ✅ Full conversation history preserved
- ✅ Seamless transition between providers
- ✅ Plan workflow (Proceed/Ignore) works with any provider

---

### Segment 6: Summary & Reflection (45 seconds)

**What to Say:**
> "That's Chartsmith with Vercel AI SDK. We've migrated from a custom Anthropic implementation to a modern, multi-provider architecture—while preserving all 27+ existing UI features.
>
> Beyond the migration, we added new capabilities: a **chart validation agent** that runs Helm lint, template validation, and kube-score—then has the AI interpret issues and suggest fixes. And **live provider switching**, so you can change models mid-conversation without losing context.
>
> AI tools significantly accelerated this project. I used CodeLayer—a Claude Code harness—for deep codebase research and architectural planning. Cursor handled the implementation, letting me iterate quickly on the adapter pattern and tool integrations.
>
> The biggest learning? You don't have to rewrite everything. The adapter pattern let me swap the transport layer without touching the UI—reducing what could have been 150+ hours of work to about 40 hours.
>
> Thanks for watching!"

**What to Show:**
- Quick pan of the completed chart in file explorer
- Final chat showing conversation across providers

**Key Points:**
- ✅ Full AI SDK migration complete
- ✅ Multi-provider support unlocked
- ✅ AI tools (CodeLayer, Cursor) accelerated development
- ✅ Adapter pattern = incremental modernization without full rewrite

---

## Post-Demo Checklist

- [ ] Video shows all required elements:
  - [ ] Chart creation with streaming
  - [ ] Page refresh showing state persistence
  - [ ] Proceed/Ignore buttons appearing
  - [ ] Validation results with severity colors
  - [ ] AI interpreting validation issues
  - [ ] Provider switching in dropdown
  - [ ] Conversation preserved after switch
- [ ] Video is ~5 minutes
- [ ] Audio quality is good
- [ ] Screen recording captures all details

---

## Troubleshooting Tips

### If Proceed/Ignore buttons don't appear:
- Wait ~500ms after streaming completes (PR4.0 refetch delay)
- Check browser console for errors
- Verify `responsePlanId` is being set in the database

### If validation fails:
- Ensure chart files exist in workspace
- Check Go backend logs for tool execution
- Verify helm, kube-score are installed

### If provider switching doesn't work:
- Verify multiple API keys are configured
- Check `OPENROUTER_API_KEY` or individual provider keys
- Check browser console for provider errors

### If streaming doesn't work:
- Check `NEXT_PUBLIC_USE_AI_SDK_CHAT=true` is set
- Verify API keys are configured
- Check browser console for errors

---

## Key Talking Points

### PR4.0 New Features

1. **Chart Validation Agent**
   - Runs helm lint → helm template → kube-score pipeline
   - Returns structured results with severity levels
   - AI interprets and provides fix suggestions

2. **Live Provider Switching**
   - Dropdown to switch providers mid-conversation
   - Conversation history fully preserved
   - Supports Claude, GPT-4, and other OpenRouter models

3. **Message Refetch Fix**
   - Polls for `responsePlanId` after streaming
   - Fixes missing Proceed/Ignore buttons
   - ~500ms delay then buttons appear

### Technical Highlights
- **Vercel AI SDK**: Standardized patterns, multi-provider support
- **Tool Integration**: 6 tools (including new validateChart)
- **State Management**: Full persistence across refresh
- **Plan Workflow**: Proceed/Ignore for all file changes

---

## Alternative: Extended Demo (7 minutes)

If you have more time, add these segments:

### Extended Segment: Code Walkthrough (2 minutes)

**What to Say:**
> "Let me show you the key code changes that power these features."

**What to Show:**

#### Adapter Pattern (`hooks/useAISDKChatAdapter.ts`)
- Show how AI SDK `useChat` bridges to existing UI
- Point out provider/model state management
- Show message format conversion

#### Validation Tool (`lib/tools/validation.ts`)
- Show `validateChart` tool definition
- Pipeline execution: lint → template → kube-score
- Structured result format

#### Live Provider Switcher (`components/LiveProviderSwitcher.tsx`)
- Dropdown component in chat header
- Provider state passed to adapter
- Seamless switching logic

### Extended Segment: Test Results (30 seconds)

**What to Show:**
```bash
cd chartsmith-app
npm run test:unit

# Results:
# Test Suites: 10 passed, 10 total
# Tests:       125 passed, 125 total
```

---

**Good luck with your demo! 🎬**
