# Chartsmith AI SDK Migration

This repository contains my work on migrating Chartsmith's chat system from a custom Go/Anthropic backend to the Vercel AI SDK with multi-provider support.

## Original Repository

**Chartsmith** by Replicated - An AI-powered Helm chart development assistant.  
Original codebase: Private Replicated repository (cloned to `./chartsmith/`)

## What I Built

Migrated Chartsmith's entire chat system to use the Vercel AI SDK while preserving all existing functionality:

### Before (Original Architecture)
```
React Chat → Server Actions → PostgreSQL queue → Go Workers → Anthropic SDK → 3 Tools
```

### After (New Architecture)
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              REACT FRONTEND                                  │
│  ┌─────────────────┐              ┌─────────────────┐                       │
│  │   Chat UI       │              │  Jotai Atoms    │◄─── useCentrifugo     │
│  │   Components    │◄────────────►│  (State Mgmt)   │     (WebSocket)       │
│  └────────┬────────┘              └─────────────────┘          ▲            │
└───────────┼─────────────────────────────────────────────────────┼────────────┘
            │ useAISDKChatAdapter                                 │
            ▼                                                     │
┌─────────────────────────────────────────────────────────────────┼────────────┐
│                     NEXT.JS + VERCEL AI SDK LAYER               │            │
│                                                                 │            │
│  ┌──────────────────────────────────────────────────────────────┼─────────┐  │
│  │                      /api/chat Route                         │         │  │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┴───────┐ │  │
│  │  │ Intent Classify │───►│   streamText    │───►│  6 TypeScript Tools │ │  │
│  │  │ (plan/proceed/  │    │   (AI SDK v5)   │    │  with Go HTTP calls │ │  │
│  │  │  ai-sdk/render) │    └────────┬────────┘    └─────────────────────┘ │  │
│  │  └─────────────────┘             │                                     │  │
│  └──────────────────────────────────┼─────────────────────────────────────┘  │
│                                     │                                        │
│                                     ▼                                        │
│                          ┌─────────────────┐                                 │
│                          │   OpenRouter    │  ◄── Multi-Provider LLM         │
│                          │  Claude Sonnet  │      (Switch models live!)      │
│                          │  GPT-4o, etc.   │                                 │
│                          └─────────────────┘                                 │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     │ HTTP (Tool Execution)
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           GO HTTP API LAYER                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  /api/tools/editor  │  /api/validate  │  /api/plan/create-from-tools    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│        ┌────────────────────────────┴────────────────────────────┐           │
│        ▼                                                         ▼           │
│  ┌─────────────────┐                                    ┌─────────────────┐  │
│  │  Plan Workflow  │                                    │   Centrifugo    │──┼──► WebSocket
│  │  (review→apply) │                                    │   (Real-time)   │  │
│  └────────┬────────┘                                    └─────────────────┘  │
└───────────┼──────────────────────────────────────────────────────────────────┘
            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              POSTGRESQL                                       │
│  workspace_plan (buffered_tool_calls)  │  workspace_chat  │  workspace_file  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Key Deliverables

| PR | Feature | Status |
|----|---------|--------|
| PR1 | AI SDK chat integration | ✅ Complete |
| PR1.5 | Tool support via Go HTTP | ✅ Complete |
| PR2.0 | Main workspace integration with adapter pattern | ✅ Complete |
| PR3.0 | Intent classification + Plan workflow parity | ✅ Complete |
| PR4 | Chart validation tool + Live provider switching | ✅ Complete |

## Architecture Overview

See [docs/CURRENT_ARCHITECTURE.md](./docs/CURRENT_ARCHITECTURE.md) for the full architecture documentation.

**Key Innovations:**
- **Adapter Pattern**: Bridged AI SDK's `useChat` to existing React components without rewriting UI
- **TypeScript Tools → Go HTTP**: Best of both worlds - TS flexibility + Go performance
- **Multi-Provider LLM**: OpenRouter enables Claude, GPT-4, Mistral switching at runtime
- **Intent Classification**: Smart routing (plan vs execute vs conversational)
- **Plan Workflow**: Buffer tool calls → user review → execute (matches legacy behavior)

## Setup + Run

### Prerequisites
- Node.js 18+
- Go 1.21+
- PostgreSQL 14+
- Docker (for Centrifugo)

### Environment Variables

```bash
# Required - at least one LLM provider
OPENROUTER_API_KEY=sk-or-v1-xxxxx    # Recommended: unified gateway
ANTHROPIC_API_KEY=sk-ant-xxxxx       # Direct Anthropic access
OPENAI_API_KEY=sk-xxxxx              # Direct OpenAI access

# Backend
GO_BACKEND_URL=http://localhost:8080
NEXT_PUBLIC_USE_AI_SDK_CHAT=true     # Enable AI SDK (default)

# Database
DB_URI=postgres://user:pass@localhost:5432/chartsmith
```

### Running Locally

```bash
# Terminal 1: Start Go backend
cd chartsmith
make run

# Terminal 2: Start Next.js frontend
cd chartsmith/chartsmith-app
npm install
npm run dev

# Terminal 3: Start Centrifugo (real-time)
docker run -p 8000:8000 centrifugo/centrifugo
```

Open http://localhost:3000

### Running Tests

```bash
# Unit tests
cd chartsmith/chartsmith-app
npm test

# E2E tests
npx playwright test

# Go tests
cd chartsmith
go test ./...
```

## Technical Decisions

### Why Adapter Pattern?
Rather than rewriting the UI to use AI SDK's message format directly, I created `useAISDKChatAdapter` that:
- Converts between AI SDK `UIMessage[]` and Chartsmith `Message[]` formats
- Maps streaming states (`status`) to existing boolean flags (`isThinking`, `isStreaming`)
- Adds persona propagation, database persistence, and Centrifugo coordination

This preserved all existing UI components while enabling full AI SDK streaming.

### Why TypeScript Tools with Go HTTP?
- **TypeScript**: Better AI SDK integration, schema validation with Zod, streaming support
- **Go**: Existing file system operations, Helm CLI execution, database transactions

The hybrid approach uses Go for what it does best (deterministic operations, Helm, filesystem) while TypeScript handles AI orchestration.

### Why OpenRouter?
- Single API key for multiple providers (Claude, GPT-4, Mistral, etc.)
- Live model switching without code changes
- Cost optimization via model selection
- Fallback to direct provider APIs if needed

### Why Intent Classification?
Pre-classifying user messages (via fast Groq model) enables:
- **Off-topic filtering**: Polite decline without expensive LLM call
- **Plan vs Execute routing**: Different prompts for planning vs execution
- **Smart tool enablement**: Only enable file-modifying tools when needed

## Project Structure

```
UnchartedTerritory/
├── chartsmith/                    # Forked Chartsmith codebase
│   ├── chartsmith-app/           # Next.js frontend
│   │   ├── app/api/chat/         # AI SDK streaming endpoint
│   │   ├── hooks/                # useAISDKChatAdapter, useCentrifugo
│   │   ├── lib/ai/               # AI SDK tools, prompts, intent
│   │   └── ARCHITECTURE.md       # Detailed frontend architecture
│   └── pkg/                      # Go backend
│       ├── api/handlers/         # HTTP endpoints for tools
│       ├── llm/                  # LLM client (legacy + intent)
│       └── validation/           # Helm lint, template, kube-score
├── docs/
│   ├── CURRENT_ARCHITECTURE.md   # Full architecture diagram
│   ├── DEMO_SCRIPT.md            # Demo walkthrough
│   └── PR*.md                    # PR completion reports
├── PRDs/                         # Product requirement documents
├── memory-bank/                  # Project context for AI assistants
└── README.md                     # This file
```

## Documentation

| Document | Description |
|----------|-------------|
| [ChartSmith_Architecture.md](./ChartSmith_Architecture.md) | Full system architecture |
| [docs/CURRENT_ARCHITECTURE.md](./docs/CURRENT_ARCHITECTURE.md) | Architecture overview with diagrams |

## License

This is a work sample. The original Chartsmith codebase is proprietary to Replicated.
