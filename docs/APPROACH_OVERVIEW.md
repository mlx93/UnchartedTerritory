# Chartsmith SDK Migration - Interview Overview

**The 2-Minute Summary for Senior Audiences**

---

## The Problem

Chartsmith is an AI-powered Helm chart creation tool with a **tightly-coupled architecture**: Go backend → PostgreSQL queue → Anthropic SDK. This meant:

- **No provider flexibility** (locked to Claude)
- **Custom streaming implementation** with high maintenance overhead
- **No standard SDK patterns** for tool calling

**Goal**: Migrate to Vercel AI SDK for multi-provider support while preserving all existing functionality.

---

## My Approach: Parallel Path + Feature Flag

Rather than a risky in-place rewrite, I built a **parallel system** that could be toggled:

```
NEXT_PUBLIC_USE_AI_SDK_CHAT=true   → New AI SDK path
NEXT_PUBLIC_USE_AI_SDK_CHAT=false  → Legacy Go worker path
```

**Why this worked:**
- Zero-risk deployment (instant rollback via env var)
- Both paths share the same database—no migration needed
- Incremental feature parity validation

---

## The Integration Strategy: Adapter Pattern

The existing UI expected message format `{ prompt, response, isComplete }`.
AI SDK provides format `{ parts[], status }`.

**Instead of rewriting 27+ UI components**, I created a single adapter hook:

```
useChat (AI SDK) → useAISDKChatAdapter → Existing Chat UI
```

This preserved **all existing features** (file explorer, plan approval, persona switching) while adding new capabilities.

**Result**: 8 hours vs. estimated 113 hours for full UI rebuild (75% effort reduction).

---

## Feature Parity: Tool Buffering Pattern

The biggest challenge was preserving the **plan approval workflow** where users review changes before execution.

**Legacy path**: Go workers regenerate content on "Proceed" click.
**New path**: Can't regenerate—AI SDK streams once.

**Solution: Buffer destructive tool calls**

1. AI calls `textEditor({ command: "create", path: "..." })`
2. Tool intercepts, buffers the call, returns `{ success: true, buffered: true }`
3. AI continues explaining what it will do
4. On "Proceed" click → execute buffered calls
5. On "Ignore" → discard buffer

Same UX, different transport.

---

## Architecture Summary

| Layer | Before | After |
|-------|--------|-------|
| LLM Calls | Go + Anthropic SDK | Node + AI SDK + OpenRouter |
| Tool Communication | Inline Go functions | HTTP endpoints (11 total) |
| Streaming | Custom Centrifugo | AI SDK `streamText()` + Centrifugo coordination |
| Provider | Anthropic only | Anthropic, OpenAI, OpenRouter (live switching) |

**Key numbers:**
- 6 AI SDK tools implemented
- 11 Go HTTP endpoints
- ~7,000 lines of new code
- 116 tests passing

---

## Key Technical Decisions

| Decision | Why |
|----------|-----|
| **HTTP over PostgreSQL queue** | AI SDK tools need synchronous responses; queue is async |
| **Adapter over rewrite** | Preserve battle-tested UI; reduce risk and timeline |
| **Tool buffering over immediate execution** | Match existing approval workflow UX |
| **Groq for intent classification** | Fast (70B model), cheap, enables smart routing |

---

## What I'd Highlight in 60 Seconds

> "I migrated Chartsmith's chat system from a custom Anthropic integration to the Vercel AI SDK. The key insight was building a **parallel path with feature flags** for safe rollout, and using an **adapter pattern** to preserve 27+ existing UI features without rewriting them. The hardest problem was maintaining the plan approval workflow—I solved it by **buffering destructive tool calls** during streaming and executing them on user approval. This gave us multi-provider support and live model switching while maintaining zero-downtime deployment capability."

---

## Lessons Learned

1. **Adapter > Rewrite** when UI already works
2. **Feature flags** turn scary migrations into safe experiments  
3. **Evidence-based research** (15 false assumptions found before coding)
4. **TypeScript type definitions** are more reliable than docs for AI SDK v5
