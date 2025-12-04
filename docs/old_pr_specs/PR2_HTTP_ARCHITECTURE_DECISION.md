# PR2 HTTP Architecture Decision: Deep Dive

**Status**: DECIDED - Option A (HTTP Endpoint)
**Date**: 2025-12-01
**Impact**: PR2 tool calling implementation

---

## The Core Question

> "Chartsmith uses PostgreSQL LISTEN/NOTIFY + Centrifugo everywhere. Adding HTTP could signal I don't understand the existing architecture. Does it violate requirements?"

---

## Decision Context

PR2 adds an HTTP endpoint (`POST /api/validate`) to the Go backend for the `validateChart` tool. Three options were evaluated:

| Option | Approach | Sync? | New Pattern? |
|--------|----------|-------|--------------|
| **A (Chosen)** | HTTP endpoint in Go | Yes | Yes |
| B | PostgreSQL queue (async) | No | No |
| C | Next.js spawns Go CLI | Yes | Partial |

---

## Why HTTP is the Right Choice

### 1. PR1 Already Establishes HTTP as the Pattern

PR1 creates a **NEW parallel chat system** that uses HTTP:

```
┌─────────────────┐     useChat Hook    ┌─────────────────┐
│   NEW Chat UI   │ ───────────────────►│  NEW /api/chat  │ ──► OpenRouter
│  (AI SDK-based) │ ◄───────────────────│   (streamText)  │
│  @ai-sdk/react  │   Data Stream       │   HTTP/SSE      │
└─────────────────┘                     └─────────────────┘
```

**This is already a departure from PostgreSQL+Centrifugo** for the new chat flow. Extending HTTP to the Go backend is consistent with this pattern.

### 2. AI SDK Tools Require Synchronous Execute

Vercel AI SDK tool definition:
```typescript
execute: async (input) => {
  // MUST return a result synchronously (Promise resolves to result)
  // Cannot return "wait for WebSocket" or "poll for result"
  const response = await fetch('/api/validate', { ... });
  return response.json();
}
```

**Option B (queue pattern) doesn't work** because:
- Tool execute() must return a result
- PostgreSQL LISTEN/NOTIFY is async - no direct result
- Would require polling/WebSocket waiting in the tool
- Adds complexity, not reduces it

### 3. The Existing Pattern Wasn't Designed for Tool Calling

Current architecture handles:
- Long-running LLM operations (10+ seconds)
- Fire-and-forget work items
- Progressive updates via Centrifugo

Validation is different:
- Quick operation (1-3 seconds)
- Needs immediate result for AI interpretation
- Tool calling expects request-response pattern

### 4. It's Isolated and Additive

The HTTP endpoint only affects validation. Existing features continue to use PostgreSQL+Centrifugo:
- Chat message handling → unchanged
- Plan execution → unchanged
- Render streaming → unchanged
- File updates → unchanged

### 5. The Spec Permits It

> "The existing Go backend will remain, but may need API adjustments"

Adding an HTTP API is an adjustment. The Go backend remains and gains new capability.

---

## Arguments Against HTTP (And Why They Don't Apply)

| Concern | Counter-Argument |
|---------|------------------|
| "Diverges from existing architecture" | PR1 already diverges with AI SDK HTTP streaming |
| "Signals misunderstanding" | Actually shows understanding of AI SDK requirements |
| "Requires additional port" | Minimal operational impact (8080 alongside 3000) |
| "Could use existing pattern" | Existing pattern is async, tools need sync |

---

## Alternative Analysis: Could We Make Queue Pattern Work?

**Option B Implementation (Async)**:
```typescript
execute: async (input) => {
  // 1. Enqueue validation job
  const jobId = await enqueueValidation(input);

  // 2. Poll for result (ugly, slow, complex)
  for (let i = 0; i < 30; i++) {
    await sleep(1000);
    const result = await checkValidationStatus(jobId);
    if (result) return result;
  }
  throw new Error('Validation timeout');
}
```

**Problems**:
- 30 HTTP requests instead of 1
- Artificial delay (polling interval)
- Complex error handling
- Race conditions with Centrifugo
- Worse UX (not "instant")

**HTTP is objectively simpler and better for this use case.**

---

## Architecture Comparison

### Before PR1 (Current)
```
Frontend → Server Action → PostgreSQL → Go Worker → Anthropic → Centrifugo → Frontend
```

### After PR1 (NEW Parallel System)
```
NEW Chat → useChat → /api/chat → OpenRouter → Data Stream → Frontend

(Existing system unchanged, runs in parallel)
```

### After PR2 (Tool Calling Added)
```
NEW Chat → useChat → /api/chat → OpenRouter
                          ↓ (tool call)
                    validateChart tool
                          ↓ (HTTP POST)
                    Go /api/validate → helm/kube-score
                          ↓ (JSON response)
                    Tool result rendered
```

**The HTTP endpoint is a natural extension of PR1's pattern, not a contradiction of it.**

---

## Implementation Details

### Go Backend Changes

**New Files**:
```
pkg/
├── api/
│   └── validate.go       # HTTP handler for POST /api/validate
└── validation/
    ├── pipeline.go       # Orchestration (lint → template → kube-score)
    ├── helm.go           # helm lint, helm template execution
    ├── kubescore.go      # kube-score execution
    ├── parser.go         # Output parsing utilities
    └── types.go          # ValidationRequest, ValidationResult types
```

**Modified Files**:
```
cmd/run.go                # Start HTTP server alongside PostgreSQL listener
```

### Environment Variables

```env
GO_BACKEND_URL=http://localhost:8080  # Frontend calls this
GO_HTTP_PORT=8080                      # Go HTTP server port (optional, default 8080)
```

### Frontend Tool Definition

```typescript
// lib/ai/tools/validateChart.ts
export const validateChart = tool({
  description: "Validate a Helm chart...",
  inputSchema: z.object({
    chartPath: z.string(),
    values: z.record(z.any()).optional(),
    strictMode: z.boolean().optional(),
    kubeVersion: z.string().optional()
  }),
  execute: async (input) => {
    const response = await fetch(`${process.env.GO_BACKEND_URL}/api/validate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(input)
    });
    if (!response.ok) throw new Error(`Validation failed: ${response.statusText}`);
    return response.json();
  }
});
```

---

## Final Verdict

**HTTP (Option A) is the correct choice** because:
1. Aligns with AI SDK's synchronous tool model
2. PR1 already establishes HTTP as the new pattern
3. Simple, testable, maintainable
4. Isolated change - doesn't affect existing features
5. The spec explicitly permits "API adjustments"

The concerns about "diverging from architecture" are addressed by recognizing that PR1 already establishes HTTP as the pattern for the new AI SDK flow. PR2 is extending that pattern, not contradicting it.
