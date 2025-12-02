# Chartsmith Current State Analysis

## Overview

Chartsmith is an AI-powered tool by Replicated that helps developers build better Helm charts. This document analyzes the current architecture based on the project specification and available documentation.

---

## System Architecture

### High-Level Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
├─────────────────────────┬───────────────────────────────────────┤
│   Next.js Web App       │      VS Code Extension                │
│   (TypeScript/React)    │      (TypeScript)                     │
│   - Custom Chat UI      │      - Token-based Auth               │
│   - Manual streaming    │      - Bearer token header            │
│   - @anthropic-ai/sdk   │      - /api/auth/status endpoint      │
└─────────────┬───────────┴───────────────────┬───────────────────┘
              │                               │
              ▼                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API LAYER                                 │
│                     Next.js API Routes                           │
│   - /api/chat (chat completions)                                │
│   - /api/auth/status (token validation)                         │
│   - Custom streaming protocol                                    │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        BACKEND LAYER                             │
│                        Go Backend                                │
│   - Helm chart processing                                        │
│   - File system operations                                       │
│   - Chart validation logic                                       │
│   - LLM integration via @anthropic-ai/sdk                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     EXTERNAL SERVICES                            │
│   - Anthropic Claude API                                         │
│   - Artifact Hub (dependency fetching)                          │
│   - Helm CLI (chart operations)                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Current LLM Integration

### Frontend (chartsmith-app)
- **SDK**: `@anthropic-ai/sdk` (direct Anthropic SDK)
- **Pattern**: Custom chat UI components with manual message handling
- **Streaming**: Manual implementation of streaming protocol
- **State Management**: Custom state management for chat interactions

### Backend (Go)
- **SDK**: Anthropic SDK for Go (or HTTP calls)
- **Pattern**: Direct API integration
- **Streaming**: Custom streaming implementation

### Pain Points (per project spec)
1. Custom chat UI components require significant maintenance
2. Manual message handling across frontend and backend
3. Custom streaming protocol implementation
4. State management complexity
5. Tight coupling to Anthropic provider

---

## Directory Structure (Inferred)

```
chartsmith/
├── chartsmith-app/           # Next.js frontend application
│   ├── ARCHITECTURE.md       # Frontend architecture docs
│   ├── src/
│   │   ├── app/             # Next.js App Router
│   │   ├── components/      # React components (including chat UI)
│   │   ├── hooks/           # Custom React hooks
│   │   ├── lib/             # Utilities and SDK integrations
│   │   └── api/             # API route handlers
│   ├── package.json
│   └── tsconfig.json
│
├── cmd/                      # Go application entry points
├── pkg/                      # Go packages
│   ├── api/                 # API handlers
│   ├── chat/                # Chat/LLM logic
│   ├── helm/                # Helm chart operations
│   └── validation/          # Chart validation
│
├── test_chart/              # Test chart fixtures
├── testdata/                # Test data files
│
├── ARCHITECTURE.md          # Root architecture docs
├── CONTRIBUTING.md          # Development setup
├── go.mod                   # Go dependencies
├── go.sum
└── README.md
```

---

## Key Functionality

### 1. Chart Creation
- Users describe what they want via chat interface
- AI generates Helm chart templates
- Iterative refinement through conversation

### 2. Chart Import & Improvement
- Import existing Helm charts
- AI analyzes and suggests improvements
- Best practices enforcement

### 3. Dependency Management
- Integration with Artifact Hub
- Automatic dependency resolution
- Version compatibility checks

### 4. Environment Compatibility
- Multi-environment support
- Configuration customization
- Security updates integration

---

## Authentication Flow (VS Code Extension)

```
┌──────────────┐     1. Click Login     ┌──────────────┐
│   VS Code    │ ───────────────────────▶│   Browser    │
│  Extension   │                         │  Auth Page   │
└──────────────┘                         └──────┬───────┘
       ▲                                        │
       │                                        │ 2. Authenticate
       │                                        ▼
       │                                 ┌──────────────┐
       │    4. Store & use token         │   Chartsmith │
       │◀────────────────────────────────│   Backend    │
       │                                 └──────┬───────┘
       │                                        │
       │         3. Generate extension token    │
       └────────────────────────────────────────┘

Token validation: GET /api/auth/status (Bearer token header)
```

---

## Test Infrastructure

### Test Locations
- `test_chart/` - Sample Helm charts for testing
- `testdata/` - Test fixtures and data files

### Test Types (Expected)
- Go unit tests (`*_test.go`)
- TypeScript/Jest tests (frontend)
- Integration tests for chat flows
- Helm chart validation tests

---

## Current Limitations

1. **Single Provider Lock-in**: Only Anthropic Claude supported
2. **Custom Streaming**: Non-standard streaming implementation
3. **State Management**: Scattered across components
4. **Maintenance Burden**: Significant custom code for chat functionality
5. **No Tool Calling Standard**: Custom implementation for any tool use

---

## Migration Opportunities

### Vercel AI SDK Benefits
| Current State | With AI SDK |
|--------------|-------------|
| Custom `useChat` implementation | `useChat` hook from `@ai-sdk/react` |
| Manual streaming protocol | Built-in `streamText` with optimized streaming |
| Anthropic-only | Multi-provider via unified API |
| Custom state management | SDK handles message state, loading, errors |
| No standardized tool calling | Built-in tool calling with `tool()` helper |

### Provider Flexibility
With Vercel AI SDK + OpenRouter:
- Access to 300+ models via single API key
- Easy provider switching without code changes
- Fallback provider support
- Cost optimization across providers

---

## References

- [Vercel AI SDK Docs](https://ai-sdk.dev/docs)
- [OpenRouter AI SDK Provider](https://github.com/OpenRouterTeam/ai-sdk-provider)
- [Chartsmith GitHub](https://github.com/replicatedhq/chartsmith)
