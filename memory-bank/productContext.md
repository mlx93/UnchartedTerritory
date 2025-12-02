# Chartsmith Product Context

**Purpose**: Explain why this migration exists, the problems it solves, and the user experience goals.

---

## Why This Migration Exists

### Current Pain Points

1. **High Maintenance Burden**
   - Custom streaming and state logic requires ongoing attention
   - Manual message handling across frontend and backend
   - Custom implementations throughout the stack

2. **Provider Inflexibility**
   - Tight coupling to Anthropic (Go backend does all heavy lifting)
   - Switching LLM providers requires significant refactoring
   - No easy way to try different models

3. **Inconsistent Patterns**
   - Non-standard implementation creates onboarding friction
   - Custom chat UI components diverge from modern patterns
   - State management in Jotai atoms (custom, not AI SDK)

4. **Limited Optimization**
   - Missing streaming and state management optimizations
   - Custom streaming via PostgreSQL work queue + Centrifugo
   - No standardized tool calling in frontend

---

## Problems Being Solved

### For Developers

| Problem | Solution |
|---------|----------|
| Maintaining custom streaming logic | AI SDK handles streaming automatically |
| Manual message state management | `useChat` hook manages message state |
| Anthropic-only integration | OpenRouter provides 300+ models via single API |
| Custom tool calling format | AI SDK `tool()` helper standardizes tools |
| Complex error handling | SDK provides consistent error patterns |

### For Users

| Problem | Solution |
|---------|----------|
| Single AI model choice | Provider selector enables model switching |
| Slow iteration on fixes | Faster development cycle with SDK patterns |
| Limited extensibility | Easy to add new tools and providers |

---

## User Experience Goals

### Chat Experience
- **Seamless streaming**: Text appears smoothly as it generates
- **Responsive UI**: < 100ms for user interactions
- **Clear status indicators**: Users know when AI is thinking/streaming
- **Error recovery**: Graceful handling with retry options

### Provider Selection
- **Easy model switching**: Dropdown to select models
- **Live switching (PR2)**: Change models mid-conversation
- **Visual feedback**: Clear indicator of current model

### Tool Integration
- **Natural invocation**: AI recognizes when to use tools
- **Progress indication**: Users see tool execution status
- **Rich results**: Formatted display of tool outputs

### Chart Creation Flow
- **Conversational creation**: "Create a nginx chart" just works
- **Guided assistance**: AI explains what it's doing
- **Visual feedback**: See chart files being created

---

## Feature Parity Requirements

All existing capabilities must continue working:

| Feature | Current Location | After Migration |
|---------|------------------|-----------------|
| Chat streaming | Centrifugo WebSocket | AI SDK Data Stream |
| text_editor tool | Go backend | AI SDK tool → Go HTTP |
| Subchart version lookup | Go backend | AI SDK tool → Go HTTP |
| K8s version info | Go backend | AI SDK tool → Go HTTP |
| Chart creation | Go workers | AI SDK tool → Go HTTP |
| File context | Jotai atoms | AI SDK context |

---

## New Capabilities Added

### PR1: Foundation
- New `/api/chat` route using Vercel AI SDK
- `useChat` hook for React integration
- Provider selection (GPT-4o, Claude 3.5 Sonnet)
- Standard HTTP streaming

### PR1.5: Migration & Parity
- 6 AI SDK tools (createChart, getChartContext, updateChart, textEditor, latestSubchartVersion, latestKubernetesVersion)
- Go HTTP server for tool execution
- Shared LLM client library
- System prompt migration

### PR2: Validation Agent
- Chart validation tool (helm lint, template, kube-score)
- Natural language validation feedback
- Best practices scoring
- Live provider switching mid-conversation

---

## User Journeys

### Creating a New Chart

```
User: "Create a nginx deployment chart"
     ↓
AI recognizes creation intent
     ↓
AI invokes createChart tool
     ↓
Go creates workspace with Chart.yaml, values.yaml, templates/
     ↓
AI confirms creation with details
     ↓
User sees new chart in workspace
```

### Validating a Chart (PR2)

```
User: "Validate my chart"
     ↓
AI invokes validateChart tool
     ↓
Go runs helm lint → helm template → kube-score
     ↓
Results returned as structured data
     ↓
AI explains issues in plain language
     ↓
User sees severity-coded issue list
     ↓
AI offers to help fix issues
```

### Switching Providers (PR2)

```
User is chatting with GPT-4o
     ↓
Clicks provider dropdown (always visible)
     ↓
Selects Claude 3.5 Sonnet
     ↓
Conversation history preserved
     ↓
Next response uses new model
```

---

## Design Principles

1. **No Feature Regression**: Everything that works today must work after migration
2. **Progressive Enhancement**: Add new capabilities without breaking existing ones
3. **Minimal Go Changes in PR1**: Foundation layer first, Go modifications in PR1.5
4. **Clean Separation**: Node handles intelligence, Go handles execution
5. **Standardized Patterns**: Follow AI SDK conventions, not custom solutions

---

*This document captures why we're doing this migration and what users should experience.*

