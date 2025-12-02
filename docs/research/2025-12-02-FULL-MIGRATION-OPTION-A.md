# Full Migration Research: Option A - Complete Vercel AI SDK Integration

---
date: 2025-12-02T05:30:00Z
researcher: Claude Code
git_commit: e615830f7b0b60ecb817d762b9b4e4186f855d85
branch: main
repository: UnchartedTerritory/chartsmith
topic: "Full Migration Option A: Complete Vercel AI SDK Integration Research"
tags: [research, full-migration, option-a, chartsmith, vercel-ai-sdk, go-backend]
status: complete
last_updated: 2025-12-02
last_updated_by: Claude Code
---

## Executive Summary

This document provides comprehensive research for implementing **Option A: Full Migration** - replacing the Go backend's LLM orchestration with Vercel AI SDK while keeping Go as a tool execution service. This approach fulfills the Replicated_Chartsmith.md requirement to "Modernize the backend: Replace direct Anthropic SDK calls with AI SDK Core's unified API."

### Key Insight

The Replicated_Chartsmith.md requirements document states (Line 43):
> "The existing Go backend will remain, but may need API adjustments"

This confirms the vision: **Go stays, but its role changes from LLM orchestrator to tool execution service.**

---

## Table of Contents

1. [Current Architecture Analysis](#1-current-architecture-analysis)
2. [What Needs to Migrate](#2-what-needs-to-migrate)
3. [Target Architecture Design](#3-target-architecture-design)
4. [Go Backend Transformation](#4-go-backend-transformation)
5. [Next.js LLM Orchestration](#5-nextjs-llm-orchestration)
6. [Tool Mapping: Go to AI SDK](#6-tool-mapping-go-to-ai-sdk)
7. [System Prompts Migration](#7-system-prompts-migration)
8. [Streaming Architecture Changes](#8-streaming-architecture-changes)
9. [Work Queue Integration](#9-work-queue-integration)
10. [Implementation Phases](#10-implementation-phases)
11. [Risk Analysis](#11-risk-analysis)
12. [Code References](#12-code-references)

---

## 1. Current Architecture Analysis

### 1.1 Current LLM Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CURRENT ARCHITECTURE                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Next.js Frontend                                                                │
│  ┌──────────────────────────────────────────────────────────────────┐           │
│  │  User Input → ChatContainer.tsx → createChatMessageAction()      │           │
│  │            → INSERT workspace_chat → pg_notify() → [exits]       │           │
│  └──────────────────────────────────────────────────────────────────┘           │
│                              │                                                   │
│                              ▼ PostgreSQL LISTEN/NOTIFY                         │
│                                                                                  │
│  Go Backend (ALL LLM ORCHESTRATION)                                             │
│  ┌──────────────────────────────────────────────────────────────────┐           │
│  │  Work Queue Listener → Handler Selection                         │           │
│  │                              │                                    │           │
│  │  ┌───────────────────────────┴───────────────────────────┐       │           │
│  │  │                                                        │       │           │
│  │  ▼              ▼              ▼              ▼          │       │           │
│  │  new_intent   new_plan   execute_plan   new_conversational│       │           │
│  │     │            │            │              │            │       │           │
│  │     ▼            ▼            ▼              ▼            │       │           │
│  │  Anthropic SDK Calls (streaming)                          │       │           │
│  │  - System prompts from system.go                          │       │           │
│  │  - Tool definitions (text_editor, subchart, k8s)         │       │           │
│  │  - Agentic loops for tool execution                       │       │           │
│  │     │                                                     │       │           │
│  │     ▼                                                     │       │           │
│  │  Centrifugo → Real-time updates to client                 │       │           │
│  └──────────────────────────────────────────────────────────┘       │           │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Go LLM Files Inventory

| File | Purpose | Lines | Complexity |
|------|---------|-------|------------|
| `client.go` | Anthropic client factory | 21 | Simple |
| `conversational.go` | Chat with tools (subchart, k8s version) | 243 | Medium |
| `execute-action.go` | File editing with text_editor tool | 677 | High |
| `initial-plan.go` | Initial plan generation | ~80 | Medium |
| `plan.go` | Plan updates | ~100 | Medium |
| `execute-plan.go` | Execution orchestration | ~60 | Medium |
| `summarize.go` | Content summarization | ~120 | Low |
| `expand.go` | Prompt expansion | 54 | Simple |
| `convert-file.go` | K8s manifest conversion | ~160 | Medium |
| `cleanup-converted-values.go` | Values.yaml cleanup | ~90 | Low |
| `system.go` | 9 system prompts | 141 | Documentation |

**Total**: ~1,746 lines of LLM orchestration code that needs analysis for migration.

### 1.3 Work Queue Handlers

From `pkg/listener/start.go`:

| Channel | Purpose | Workers | Timeout |
|---------|---------|---------|---------|
| `new_intent` | Classify user intent | 5 | 10s |
| `new_summarize` | Summarize content | 5 | 10s |
| `new_plan` | Generate plan | 5 | 10s |
| `new_conversational` | Conversational chat | 5 | 10s |
| `execute_plan` | Execute file changes | 5 | 10s |
| `apply_plan` | Apply plan to workspace | 10 | 10min |
| `render_workspace` | Render Helm chart | 5 | 10s |
| `new_conversion` | K8s → Helm conversion | 5 | 10s |
| `conversion_next_file` | Convert next file | 10 | 10s |
| `conversion_normalize_values` | Normalize values.yaml | 10 | 10s |
| `conversion_simplify` | Simplify conversion | 10 | 10s |
| `publish_workspace` | Publish workspace | 20 | 5min |

**LLM-dependent handlers** (must migrate):
- `new_intent`, `new_summarize`, `new_plan`, `new_conversational`, `execute_plan`, `conversion_next_file`, `conversion_normalize_values`

**Non-LLM handlers** (stay in Go):
- `apply_plan`, `render_workspace`, `publish_workspace`

---

## 2. What Needs to Migrate

### 2.1 Migration Scope

| Component | Current Location | Target Location | Migration Type |
|-----------|-----------------|-----------------|----------------|
| LLM Client Creation | Go `pkg/llm/client.go` | Next.js `lib/ai/provider.ts` | Replace |
| System Prompts | Go `pkg/llm/system.go` | Next.js `lib/ai/prompts.ts` | Port |
| Conversational Chat | Go `pkg/llm/conversational.go` | Next.js `/api/chat/route.ts` | Rewrite |
| Plan Generation | Go `pkg/llm/plan.go`, `initial-plan.go` | Next.js `/api/chat/route.ts` | Rewrite |
| Tool Definitions | Go `pkg/llm/*.go` (inline) | Next.js `lib/ai/tools.ts` | Rewrite |
| Agentic Loops | Go `pkg/llm/execute-action.go` | Next.js AI SDK `maxSteps` | Replace |
| Streaming | Go → Centrifugo | Next.js → AI SDK Data Stream | Replace |
| Tool Execution | Go (inline with LLM) | Go HTTP API (separate) | Extract |

### 2.2 What Stays in Go

| Component | Reason |
|-----------|--------|
| `text_editor` tool execution | File system access, fuzzy matching logic |
| `helm lint/template` execution | CLI tools, Helm SDK |
| `kube-score` execution | CLI tool |
| ArtifactHub API calls | External API integration |
| K8s version lookup | External data |
| Work queue processing (non-LLM) | `apply_plan`, `render_workspace` |
| Centrifugo integration | Real-time updates |
| Database operations | File storage, workspace management |

---

## 3. Target Architecture Design

### 3.1 Option A Full Migration Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         TARGET ARCHITECTURE (OPTION A)                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Next.js Frontend + API (ALL LLM ORCHESTRATION)                                 │
│  ┌──────────────────────────────────────────────────────────────────┐           │
│  │                                                                   │           │
│  │  User Input → ChatContainer.tsx (useChat hook)                   │           │
│  │            → /api/chat POST                                       │           │
│  │                    │                                              │           │
│  │                    ▼                                              │           │
│  │            ┌───────────────────────────────────┐                 │           │
│  │            │       streamText()                 │                 │           │
│  │            │  ┌─────────────────────────────┐  │                 │           │
│  │            │  │ AI SDK Core                  │  │                 │           │
│  │            │  │ - OpenRouter/Anthropic       │  │                 │           │
│  │            │  │ - System prompts             │  │                 │           │
│  │            │  │ - Tool definitions           │  │                 │           │
│  │            │  │ - maxSteps for agentic loops │  │                 │           │
│  │            │  └─────────────────────────────┘  │                 │           │
│  │            │             │                      │                 │           │
│  │            │             ▼ Tool Calls          │                 │           │
│  │            │  ┌─────────────────────────────┐  │                 │           │
│  │            │  │ Tool Execution              │  │                 │           │
│  │            │  │ - text_editor → Go HTTP API │  │                 │           │
│  │            │  │ - subchart_version → Go API │  │                 │           │
│  │            │  │ - validateChart → Go API    │  │                 │           │
│  │            │  └─────────────────────────────┘  │                 │           │
│  │            │             │                      │                 │           │
│  │            │             ▼                      │                 │           │
│  │            │   AI SDK Data Stream Response     │                 │           │
│  │            └───────────────────────────────────┘                 │           │
│  │                                                                   │           │
│  │  Database Updates → Centrifugo → Real-time UI updates           │           │
│  │                                                                   │           │
│  └──────────────────────────────────────────────────────────────────┘           │
│                              │                                                   │
│                              ▼ HTTP API Calls (tools only)                      │
│                                                                                  │
│  Go Backend (TOOL EXECUTION SERVICE ONLY)                                       │
│  ┌──────────────────────────────────────────────────────────────────┐           │
│  │                                                                   │           │
│  │  NEW HTTP Server (net/http or Gin)                               │           │
│  │  ┌─────────────────────────────────────────────────────┐         │           │
│  │  │ POST /api/tools/text-editor                          │         │           │
│  │  │   - view, str_replace, create commands               │         │           │
│  │  │   - Fuzzy string matching                            │         │           │
│  │  │                                                       │         │           │
│  │  │ POST /api/tools/subchart-version                      │         │           │
│  │  │   - ArtifactHub lookup                               │         │           │
│  │  │                                                       │         │           │
│  │  │ POST /api/tools/kubernetes-version                    │         │           │
│  │  │   - K8s version info                                 │         │           │
│  │  │                                                       │         │           │
│  │  │ POST /api/tools/validate-chart (PR2)                  │         │           │
│  │  │   - helm lint, helm template, kube-score             │         │           │
│  │  └─────────────────────────────────────────────────────┘         │           │
│  │                                                                   │           │
│  │  Existing Work Queue Handlers (non-LLM)                          │           │
│  │  - apply_plan, render_workspace, publish_workspace               │           │
│  │                                                                   │           │
│  └──────────────────────────────────────────────────────────────────┘           │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Key Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| **All LLM calls in Next.js** | Fulfills requirement, enables multi-provider support |
| **Go becomes HTTP tool service** | Preserves complex Go logic (fuzzy matching, Helm SDK) |
| **Keep Centrifugo** | Non-LLM real-time updates still needed |
| **Keep work queue for non-LLM** | `apply_plan`, `render_workspace` don't need LLM |
| **AI SDK Data Stream for chat** | Standard streaming, replaces custom implementation |

---

## 4. Go Backend Transformation

### 4.1 New HTTP API Endpoints

The Go backend needs a new HTTP server with these endpoints:

```go
// pkg/api/server.go (NEW FILE)

package api

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func StartAPIServer(port string) error {
    r := gin.Default()

    // Tool execution endpoints
    tools := r.Group("/api/tools")
    {
        tools.POST("/text-editor", handleTextEditor)
        tools.POST("/subchart-version", handleSubchartVersion)
        tools.POST("/kubernetes-version", handleKubernetesVersion)
        tools.POST("/validate-chart", handleValidateChart) // PR2
    }

    // Health check
    r.GET("/health", func(c *gin.Context) {
        c.JSON(200, gin.H{"status": "ok"})
    })

    return r.Run(":" + port)
}
```

### 4.2 Text Editor Tool Endpoint

Extract existing logic from `execute-action.go`:

```go
// pkg/api/handlers/text_editor.go (NEW FILE)

package handlers

import (
    "github.com/gin-gonic/gin"
    "github.com/replicatedhq/chartsmith/pkg/llm"
)

type TextEditorRequest struct {
    Command string `json:"command"` // view, str_replace, create
    Path    string `json:"path"`
    Content string `json:"content"` // Current file content
    OldStr  string `json:"old_str,omitempty"`
    NewStr  string `json:"new_str,omitempty"`
}

type TextEditorResponse struct {
    Success bool   `json:"success"`
    Content string `json:"content,omitempty"`
    Error   string `json:"error,omitempty"`
}

func HandleTextEditor(c *gin.Context) {
    var req TextEditorRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, TextEditorResponse{Error: err.Error()})
        return
    }

    switch req.Command {
    case "view":
        if req.Content == "" {
            c.JSON(200, TextEditorResponse{
                Success: false,
                Error: "File does not exist. Use create instead.",
            })
            return
        }
        c.JSON(200, TextEditorResponse{
            Success: true,
            Content: req.Content,
        })

    case "str_replace":
        // Use existing PerformStringReplacement with fuzzy matching
        newContent, found, err := llm.PerformStringReplacement(
            req.Content, req.OldStr, req.NewStr,
        )
        if err != nil || !found {
            c.JSON(200, TextEditorResponse{
                Success: false,
                Error: "String to replace not found in file",
            })
            return
        }
        c.JSON(200, TextEditorResponse{
            Success: true,
            Content: newContent,
        })

    case "create":
        if req.Content != "" {
            c.JSON(200, TextEditorResponse{
                Success: false,
                Error: "File already exists. Use view and str_replace instead.",
            })
            return
        }
        c.JSON(200, TextEditorResponse{
            Success: true,
            Content: req.NewStr,
        })
    }
}
```

### 4.3 Subchart Version Endpoint

```go
// pkg/api/handlers/subchart.go (NEW FILE)

package handlers

import (
    "github.com/gin-gonic/gin"
    "github.com/replicatedhq/chartsmith/pkg/recommendations"
)

type SubchartVersionRequest struct {
    ChartName string `json:"chart_name" binding:"required"`
}

type SubchartVersionResponse struct {
    Version string `json:"version,omitempty"`
    Error   string `json:"error,omitempty"`
}

func HandleSubchartVersion(c *gin.Context) {
    var req SubchartVersionRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, SubchartVersionResponse{Error: err.Error()})
        return
    }

    version, err := recommendations.GetLatestSubchartVersion(req.ChartName)
    if err != nil {
        if err == recommendations.ErrNoArtifactHubPackage {
            c.JSON(200, SubchartVersionResponse{Version: "?"})
            return
        }
        c.JSON(500, SubchartVersionResponse{Error: err.Error()})
        return
    }

    c.JSON(200, SubchartVersionResponse{Version: version})
}
```

### 4.4 Kubernetes Version Endpoint

```go
// pkg/api/handlers/kubernetes.go (NEW FILE)

package handlers

import "github.com/gin-gonic/gin"

type KubernetesVersionRequest struct {
    SemverField string `json:"semver_field" binding:"required"` // major, minor, patch
}

type KubernetesVersionResponse struct {
    Version string `json:"version"`
}

func HandleKubernetesVersion(c *gin.Context) {
    var req KubernetesVersionRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }

    var version string
    switch req.SemverField {
    case "major":
        version = "1"
    case "minor":
        version = "1.32"
    case "patch":
        version = "1.32.1"
    default:
        version = "1.32.1"
    }

    c.JSON(200, KubernetesVersionResponse{Version: version})
}
```

### 4.5 Validate Chart Endpoint (PR2)

```go
// pkg/api/handlers/validate.go (NEW FILE)

package handlers

import (
    "github.com/gin-gonic/gin"
    "github.com/replicatedhq/chartsmith/pkg/helm"
)

type ValidateChartRequest struct {
    ChartPath  string                 `json:"chart_path" binding:"required"`
    Values     map[string]interface{} `json:"values,omitempty"`
    StrictMode bool                   `json:"strict_mode,omitempty"`
}

type ValidationResult struct {
    Tool     string   `json:"tool"`
    Success  bool     `json:"success"`
    Messages []string `json:"messages"`
}

type ValidateChartResponse struct {
    Valid   bool               `json:"valid"`
    Results []ValidationResult `json:"results"`
}

func HandleValidateChart(c *gin.Context) {
    var req ValidateChartRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }

    results := []ValidationResult{}

    // helm lint
    lintResult := helm.Lint(req.ChartPath, req.Values)
    results = append(results, ValidationResult{
        Tool:     "helm-lint",
        Success:  lintResult.Success,
        Messages: lintResult.Messages,
    })

    // helm template
    templateResult := helm.Template(req.ChartPath, req.Values)
    results = append(results, ValidationResult{
        Tool:     "helm-template",
        Success:  templateResult.Success,
        Messages: templateResult.Messages,
    })

    // kube-score (if strict mode)
    if req.StrictMode {
        scoreResult := helm.KubeScore(req.ChartPath)
        results = append(results, ValidationResult{
            Tool:     "kube-score",
            Success:  scoreResult.Success,
            Messages: scoreResult.Messages,
        })
    }

    allValid := true
    for _, r := range results {
        if !r.Success {
            allValid = false
            break
        }
    }

    c.JSON(200, ValidateChartResponse{
        Valid:   allValid,
        Results: results,
    })
}
```

---

## 5. Next.js LLM Orchestration

### 5.1 New API Route Structure

```
chartsmith-app/
├── app/
│   └── api/
│       └── chat/
│           └── route.ts           # Main AI SDK endpoint (NEW)
├── lib/
│   └── ai/                        # NEW directory
│       ├── provider.ts            # OpenRouter/provider factory
│       ├── models.ts              # Model definitions
│       ├── config.ts              # AI configuration
│       ├── prompts.ts             # System prompts (from system.go)
│       └── tools/                 # NEW directory
│           ├── index.ts           # Tool exports
│           ├── text-editor.ts     # text_editor tool
│           ├── subchart.ts        # subchart_version tool
│           ├── kubernetes.ts      # kubernetes_version tool
│           └── validate.ts        # validateChart tool (PR2)
```

### 5.2 Main Chat Route Implementation

```typescript
// app/api/chat/route.ts (NEW FILE)

import { streamText, convertToCoreMessages } from 'ai';
import { getModel } from '@/lib/ai/provider';
import { getSystemPrompt } from '@/lib/ai/prompts';
import { chartTools } from '@/lib/ai/tools';

export const maxDuration = 120; // 2 minutes for complex operations

export async function POST(req: Request) {
  const {
    messages,
    provider = 'anthropic',
    model,
    workspaceId,
    chatType = 'conversational', // 'conversational' | 'plan' | 'execute'
    context, // Workspace context (chart structure, relevant files)
  } = await req.json();

  // Build system prompt based on chat type
  const systemPrompt = getSystemPrompt(chatType, context);

  // Select tools based on chat type
  const tools = chatType === 'execute'
    ? { textEditor: chartTools.textEditor }
    : {
        subchartVersion: chartTools.subchartVersion,
        kubernetesVersion: chartTools.kubernetesVersion,
        validateChart: chartTools.validateChart,
      };

  const result = await streamText({
    model: getModel(provider, model),
    system: systemPrompt,
    messages: convertToCoreMessages(messages),
    tools,
    maxSteps: chatType === 'execute' ? 20 : 5, // More steps for file editing
    onFinish: async ({ text, toolCalls, usage }) => {
      // Save to database, update workspace
      await saveMessageToWorkspace(workspaceId, {
        role: 'assistant',
        content: text,
        toolCalls,
        usage,
      });
    },
  });

  return result.toDataStreamResponse();
}
```

### 5.3 Provider Factory

```typescript
// lib/ai/provider.ts (NEW FILE)

import { createOpenRouter } from '@openrouter/ai-sdk-provider';
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';

export type ProviderType = 'openai' | 'anthropic' | 'google';

const openrouter = createOpenRouter({
  apiKey: process.env.OPENROUTER_API_KEY!,
});

const PROVIDER_MODELS: Record<ProviderType, string> = {
  anthropic: 'anthropic/claude-3.5-sonnet',
  openai: 'openai/gpt-4o',
  google: 'google/gemini-pro',
};

export function getModel(provider: ProviderType, customModel?: string) {
  const modelId = customModel || PROVIDER_MODELS[provider];

  // If using OpenRouter (recommended for multi-provider)
  if (process.env.OPENROUTER_API_KEY) {
    return openrouter(modelId);
  }

  // Fallback to direct providers
  switch (provider) {
    case 'openai':
      return openai('gpt-4o');
    case 'anthropic':
      return anthropic('claude-3-5-sonnet-20241022');
    default:
      throw new Error(`Unsupported provider: ${provider}`);
  }
}
```

### 5.4 Tool Definitions

```typescript
// lib/ai/tools/text-editor.ts (NEW FILE)

import { tool } from 'ai';
import { z } from 'zod';

const GO_API_URL = process.env.GO_BACKEND_URL || 'http://localhost:8080';

export const textEditor = tool({
  description: `Edit files in a Helm chart. Commands:
    - view: View current file contents
    - str_replace: Replace a string in the file
    - create: Create a new file`,
  parameters: z.object({
    command: z.enum(['view', 'str_replace', 'create']),
    path: z.string().describe('File path within the chart'),
    content: z.string().optional().describe('Current file content (for context)'),
    old_str: z.string().optional().describe('String to replace'),
    new_str: z.string().optional().describe('Replacement string'),
  }),
  execute: async ({ command, path, content, old_str, new_str }) => {
    const response = await fetch(`${GO_API_URL}/api/tools/text-editor`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ command, path, content, old_str, new_str }),
    });

    const result = await response.json();

    if (!result.success) {
      return { error: result.error };
    }

    return { content: result.content };
  },
});
```

```typescript
// lib/ai/tools/subchart.ts (NEW FILE)

import { tool } from 'ai';
import { z } from 'zod';

const GO_API_URL = process.env.GO_BACKEND_URL || 'http://localhost:8080';

export const subchartVersion = tool({
  description: 'Get the latest version of a Helm subchart from ArtifactHub',
  parameters: z.object({
    chart_name: z.string().describe('The name of the subchart'),
  }),
  execute: async ({ chart_name }) => {
    const response = await fetch(`${GO_API_URL}/api/tools/subchart-version`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ chart_name }),
    });

    const result = await response.json();
    return { version: result.version || '?' };
  },
});
```

```typescript
// lib/ai/tools/kubernetes.ts (NEW FILE)

import { tool } from 'ai';
import { z } from 'zod';

const GO_API_URL = process.env.GO_BACKEND_URL || 'http://localhost:8080';

export const kubernetesVersion = tool({
  description: 'Get the latest Kubernetes version',
  parameters: z.object({
    semver_field: z.enum(['major', 'minor', 'patch'])
      .describe('Which part of the version to return'),
  }),
  execute: async ({ semver_field }) => {
    const response = await fetch(`${GO_API_URL}/api/tools/kubernetes-version`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ semver_field }),
    });

    const result = await response.json();
    return { version: result.version };
  },
});
```

```typescript
// lib/ai/tools/validate.ts (NEW FILE)

import { tool } from 'ai';
import { z } from 'zod';

const GO_API_URL = process.env.GO_BACKEND_URL || 'http://localhost:8080';

export const validateChart = tool({
  description: `Validate a Helm chart using helm lint, helm template, and kube-score.
    Returns validation results with success status and messages.`,
  parameters: z.object({
    chart_path: z.string().describe('Path to the Helm chart'),
    values: z.record(z.any()).optional().describe('Values to use for validation'),
    strict_mode: z.boolean().optional().describe('Enable kube-score validation'),
  }),
  execute: async ({ chart_path, values, strict_mode }) => {
    const response = await fetch(`${GO_API_URL}/api/tools/validate-chart`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ chart_path, values, strict_mode }),
    });

    const result = await response.json();
    return result;
  },
});
```

```typescript
// lib/ai/tools/index.ts (NEW FILE)

import { textEditor } from './text-editor';
import { subchartVersion } from './subchart';
import { kubernetesVersion } from './kubernetes';
import { validateChart } from './validate';

export const chartTools = {
  textEditor,
  subchartVersion,
  kubernetesVersion,
  validateChart,
};
```

---

## 6. Tool Mapping: Go to AI SDK

### 6.1 Tool Format Comparison

**Go (Anthropic Native)**:
```go
// From conversational.go:99-128
tools := []anthropic.ToolParam{
    {
        Name:        anthropic.F("latest_subchart_version"),
        Description: anthropic.F("Return the latest version of a subchart from name"),
        InputSchema: anthropic.F(interface{}(map[string]interface{}{
            "type": "object",
            "properties": map[string]interface{}{
                "chart_name": map[string]interface{}{
                    "type":        "string",
                    "description": "The subchart name to get the latest version of",
                },
            },
            "required": []string{"chart_name"},
        })),
    },
}
```

**AI SDK (Zod Schema)**:
```typescript
// From lib/ai/tools/subchart.ts
export const subchartVersion = tool({
  description: 'Get the latest version of a Helm subchart from ArtifactHub',
  parameters: z.object({
    chart_name: z.string().describe('The name of the subchart'),
  }),
  execute: async ({ chart_name }) => { ... },
});
```

### 6.2 Complete Tool Mapping

| Go Tool | Go File:Line | AI SDK Tool | Execute Location |
|---------|--------------|-------------|------------------|
| `text_editor_20241022` | `execute-action.go:510-532` | `textEditor` | Go HTTP API |
| `latest_subchart_version` | `conversational.go:100-113` | `subchartVersion` | Go HTTP API |
| `latest_kubernetes_version` | `conversational.go:114-128` | `kubernetesVersion` | Go HTTP API |
| (NEW - PR2) | N/A | `validateChart` | Go HTTP API |

---

## 7. System Prompts Migration

### 7.1 Prompt Inventory from system.go

| Prompt | Lines | Purpose | Migration Notes |
|--------|-------|---------|-----------------|
| `endUserSystemPrompt` | 3-18 | End-user SRE assistant | Optional - for different user role |
| `commonSystemPrompt` | 20-55 | Base prompt for all | Core - include in all |
| `chatOnlySystemPrompt` | 57-65 | Conversational chat | Add to `commonSystemPrompt` |
| `chatOnlyInstructions` | (referenced) | Chat-specific instructions | Extract from code |
| `initialPlanSystemPrompt` | 67-76 | Initial plan creation | For plan type |
| `updatePlanSystemPrompt` | 78-87 | Plan updates | For plan updates |
| `detailedPlanSystemPrompt` | 89-98 | Detailed plan with artifacts | For plan execution |
| `cleanupConvertedValuesSystemPrompt` | 100-109 | Values.yaml cleanup | For conversion |
| `executePlanSystemPrompt` | 111-120 | Single file execution | For file editing |
| `convertFileSystemPrompt` | 122-140 | K8s → Helm conversion | For conversion |

### 7.2 Migrated Prompts Implementation

```typescript
// lib/ai/prompts.ts (NEW FILE)

export type ChatType =
  | 'conversational'
  | 'initial_plan'
  | 'update_plan'
  | 'execute_plan'
  | 'convert_file'
  | 'cleanup_values';

export interface PromptContext {
  chartStructure?: string;
  relevantFiles?: Array<{ path: string; content: string }>;
  planDescription?: string;
  currentFile?: { path: string; content: string };
}

const commonSystemPrompt = `You are ChartSmith, an expert AI assistant and a highly skilled senior software developer specializing in the creation, improvement, and maintenance of Helm charts.
Your primary responsibility is to help users transform, refine, and optimize Helm charts based on a variety of inputs, including:

- Existing Helm charts that need adjustments, improvements, or best-practice refinements.

Your guidance should be exhaustive, thorough, and precisely tailored to the user's needs.
Always ensure that your output is a valid, production-ready Helm chart setup adhering to Helm best practices.
If the user provides partial information (e.g., a single Deployment manifest, a partial Chart.yaml, or just an image and port configuration), you must integrate it into a coherent chart.
Requests will always be based on an existing Helm chart and you must incorporate modifications while preserving and improving the chart's structure (do not rewrite the chart for each request).

Below are guidelines and constraints you must always follow:

<system_constraints>
  - Focus exclusively on tasks related to Helm charts and Kubernetes manifests. Do not address topics outside of Kubernetes, Helm, or their associated configurations.
  - Assume a standard Kubernetes environment, where Helm is available.
  - Do not assume any external services (e.g., cloud-hosted registries or databases) unless the user's scenario explicitly includes them.
  - Do not rely on installing arbitrary tools; you are guiding and generating Helm chart files and commands only.
  - Incorporate changes into the most recent version of files. Make sure to provide complete updated file contents.
</system_constraints>

<code_formatting_info>
  - Use 2 spaces for indentation in all YAML files.
  - Ensure YAML and Helm templates are valid, syntactically correct, and adhere to Kubernetes resource definitions.
  - Use proper Helm templating expressions ({{ ... }}) where appropriate. For example, parameterize image tags, resource counts, ports, and labels.
  - Keep the chart well-structured and maintainable.
</code_formatting_info>

<message_formatting_info>
  - Use only valid Markdown for your responses unless required by the instructions below.
  - Do not use HTML elements.
  - Communicate in plain Markdown. Inside these tags, produce only the required YAML, shell commands, or file contents.
</message_formatting_info>

NEVER use the word "artifact" in your final messages to the user.`;

const chatOnlyAddendum = `
<question_instructions>
  - You will be asked to answer a question.
  - You will be given the question and the context of the question.
  - You will be given the current chat history.
  - You will be asked to answer the question based on the context and the chat history.
  - You can provide small examples of code, but just use markdown.
</question_instructions>`;

const planAddendum = `
<testing_info>
  - The user has access to an extensive set of tools to evaluate and test your output.
  - The user will provide multiple values.yaml to test the Helm chart generation.
  - For each change, the user will run \`helm template\` with all available values.yaml and confirm that it renders into valid YAML.
  - For each change, the user will run \`helm upgrade --install --dry-run\` with all available values.yaml and confirm that there are no errors.
  - For selected changes, the user has access to and will use a tool called "Compatibility Matrix" that creates a real matrix of Kubernetes clusters such as OpenShift, RKE2, EKS, and others.
</testing_info>

NEVER use the word "artifact" in your final messages to the user. Just follow the instructions and use the text_editor tool as needed.`;

const executePlanAddendum = `
<execution_instructions>
  1. You will be asked to edit a single file for a Helm chart.
  2. You will be given the current file. If it's empty, you should create the file to meet the requirements provided.
  3. If the file is not empty, you should update the file to meet the requirements provided. In this case, provide just a patch file back.
  4. When editing an existing file, you should only edit the file to meet the requirements provided. Do not make any other changes to the file. Attempt to maintain as much of the current file as possible.
  5. You don't need to explain the change, just provide the artifact(s) in your response.
  6. Do not provide any other comments, just edit the files.
  7. Do not describe what you are going to do, just do it.
</execution_instructions>`;

const convertFileAddendum = `
<convert_file_instructions>
  - You will be given a single plain Kubernetes manifest that is part of a larger application.
  - You will be asked to convert this manifest to a helm template.
  - The template will be incorporated into a larger helm chart.
  - You will be given an existing values.yaml file to use.
  - You can re-use keys and values from the existing values.yaml file.
  - You can add new values to the values.yaml file if needed. Make sure the values don't conflict with other values.
  - Structure the values.yaml file as if there will be multiple images and it's a complex chart.
  - You may not delete or change existing keys and values from the existing values.yaml file.
  - Do not explain how to use it or provide any other instructions. Just return the values.yaml file.
  - When asked to update the values.yaml file, you MUST generate a complete unified diff patch in the standard format:
     - Start with "--- filename" and "+++ filename" headers
     - Include ONE hunk header in "@@ -lineNum,count +lineNum,count @@" format
     - Only add/remove lines should have "+" or "-" prefixes
  - When asked to convert a Kubernetes manifest, you MUST return the entire converted manifest.
  - When creating new values for the values.yaml, expect that this will be a complex chart and you should not have a very flat values.yaml schema
</convert_file_instructions>`;

export function getSystemPrompt(chatType: ChatType, context?: PromptContext): string {
  let prompt = commonSystemPrompt;

  // Add context if provided
  if (context?.chartStructure) {
    prompt += `\n\nI am working on a Helm chart that has the following structure:\n${context.chartStructure}`;
  }

  if (context?.relevantFiles?.length) {
    prompt += '\n\nRelevant files:\n';
    for (const file of context.relevantFiles) {
      prompt += `\nFile: ${file.path}\nContent:\n${file.content}\n`;
    }
  }

  if (context?.planDescription) {
    prompt += `\n\nCurrent plan:\n${context.planDescription}`;
  }

  // Add type-specific addendum
  switch (chatType) {
    case 'conversational':
      prompt += chatOnlyAddendum;
      break;
    case 'initial_plan':
    case 'update_plan':
      prompt += planAddendum;
      break;
    case 'execute_plan':
      prompt += executePlanAddendum;
      if (context?.currentFile) {
        prompt += `\n\nCurrent file (${context.currentFile.path}):\n${context.currentFile.content}`;
      }
      break;
    case 'convert_file':
      prompt += convertFileAddendum;
      break;
  }

  return prompt;
}
```

---

## 8. Streaming Architecture Changes

### 8.1 Current vs Target Streaming

| Aspect | Current (Go) | Target (AI SDK) |
|--------|--------------|-----------------|
| Protocol | Custom Centrifugo WebSocket | AI SDK Data Stream |
| Response format | Custom JSON events | Standard SSE |
| State management | Jotai atoms | `useChat` hook |
| Tool results | Streamed via Centrifugo | Inline in stream |
| Error handling | Custom error events | Standard error format |

### 8.2 Streaming Implementation

```typescript
// Frontend: components/Chat.tsx
'use client';

import { useChat } from '@ai-sdk/react';

interface ChatProps {
  workspaceId: string;
  chatType: 'conversational' | 'plan' | 'execute';
}

export function Chat({ workspaceId, chatType }: ChatProps) {
  const {
    messages,
    input,
    handleInputChange,
    handleSubmit,
    status,
    error,
    stop,
    reload,
  } = useChat({
    api: '/api/chat',
    body: {
      workspaceId,
      chatType,
    },
    onFinish: (message) => {
      // Update Jotai atoms for workspace state if needed
      // This bridges old and new state management
    },
    onError: (error) => {
      console.error('Chat error:', error);
    },
  });

  return (
    <div>
      {/* Message rendering */}
      {messages.map((message) => (
        <div key={message.id}>
          <strong>{message.role}:</strong>
          {message.parts?.map((part, i) => {
            if (part.type === 'text') {
              return <p key={i}>{part.text}</p>;
            }
            if (part.type === 'tool-result') {
              return <ToolResultDisplay key={i} result={part} />;
            }
            return null;
          })}
        </div>
      ))}

      {/* Status indicator */}
      {status === 'streaming' && <LoadingIndicator />}

      {/* Input form */}
      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Ask about Helm charts..."
          disabled={status !== 'ready'}
        />
        <button type="submit" disabled={status !== 'ready'}>
          Send
        </button>
        {status === 'streaming' && (
          <button type="button" onClick={stop}>Stop</button>
        )}
      </form>
    </div>
  );
}
```

### 8.3 Centrifugo Coexistence

**Keep Centrifugo for non-LLM events**:
- `plan-updated` - When plan status changes
- `render-stream` - When rendering completes
- `artifact-updated` - When files are saved
- `revision-created` - When new revision is created

**Remove Centrifugo for**:
- `chatmessage-updated` - Now handled by AI SDK stream

---

## 9. Work Queue Integration

### 9.1 Handler Migration Plan

| Handler | Current | After Migration |
|---------|---------|-----------------|
| `new_intent` | Go LLM call | **REMOVE** - handled in `/api/chat` |
| `new_summarize` | Go LLM call | **REMOVE** - handled in `/api/chat` |
| `new_plan` | Go LLM call | **REMOVE** - handled in `/api/chat` |
| `new_conversational` | Go LLM call | **REMOVE** - handled in `/api/chat` |
| `execute_plan` | Go LLM call | **REMOVE** - handled in `/api/chat` |
| `apply_plan` | Go file operations | **KEEP** - no LLM needed |
| `render_workspace` | Go Helm commands | **KEEP** - no LLM needed |
| `new_conversion` | Go LLM call | **MIGRATE** - complex, may need queue |
| `conversion_next_file` | Go LLM call | **MIGRATE** - complex, may need queue |
| `conversion_normalize_values` | Go LLM call | **MIGRATE** - complex, may need queue |
| `conversion_simplify` | Go LLM call | **MIGRATE** - complex, may need queue |
| `publish_workspace` | Go file operations | **KEEP** - no LLM needed |

### 9.2 Modified Work Queue Pattern

For complex multi-file operations (like K8s conversion), the work queue pattern may still be useful:

```typescript
// Option 1: Direct API calls for simple operations
// POST /api/chat → immediate response

// Option 2: Background jobs for complex operations
// POST /api/conversion/start → returns job ID
// Work queue processes files sequentially
// Each file conversion uses /api/chat internally
// Status updates via Centrifugo
```

---

## 10. Implementation Phases

### Phase 1: Foundation (Week 1-2)

1. **Go HTTP Server Setup**
   - Add HTTP server to Go backend (alongside existing work queue)
   - Implement `/api/tools/text-editor` endpoint
   - Implement `/api/tools/subchart-version` endpoint
   - Implement `/api/tools/kubernetes-version` endpoint
   - Add health check endpoint

2. **Next.js AI SDK Setup**
   - Create `lib/ai/` directory structure
   - Implement provider factory with OpenRouter
   - Migrate system prompts from Go
   - Create tool definitions

### Phase 2: Chat Migration (Week 2-3)

1. **Conversational Chat**
   - Implement `/api/chat` route
   - Migrate `new_conversational` logic
   - Update frontend to use `useChat`
   - Test streaming, tool calling

2. **Plan Generation**
   - Add plan types to `/api/chat`
   - Migrate `new_plan` logic
   - Migrate `initial_plan` logic
   - Test plan creation flow

### Phase 3: Execute Plan Migration (Week 3-4)

1. **File Editing**
   - Add `execute_plan` type to `/api/chat`
   - Wire up `textEditor` tool to Go HTTP API
   - Test file view/create/replace operations
   - Verify fuzzy matching works via HTTP

2. **Agentic Loops**
   - Configure `maxSteps` for file editing
   - Test multi-step file operations
   - Verify timeout handling

### Phase 4: Cleanup & Integration (Week 4-5)

1. **Remove Old Code**
   - Delete Go LLM orchestration files
   - Remove work queue handlers for LLM operations
   - Update work queue to only handle non-LLM jobs

2. **Testing & Validation**
   - End-to-end testing
   - Performance comparison
   - Provider switching tests

### Phase 5: K8s Conversion (Week 5-6)

1. **Conversion Flow**
   - Migrate `new_conversion` handler
   - Implement conversion state machine in Next.js
   - Use work queue for file-by-file processing
   - Each file uses `/api/chat` for LLM calls

---

## 11. Risk Analysis

### 11.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Fuzzy matching latency over HTTP | Medium | Medium | Benchmark, consider caching |
| Complex state in agentic loops | High | High | Thorough testing, fallback to simpler mode |
| Work queue race conditions | Medium | Medium | Transaction-based updates |
| Streaming reliability | Low | High | AI SDK handles retries |
| Provider switching bugs | Medium | Low | Feature flag, gradual rollout |

### 11.2 Operational Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Go HTTP server stability | Low | High | Health checks, graceful shutdown |
| Increased API latency | Medium | Medium | Connection pooling, keep-alive |
| Cost increase (multi-provider) | Medium | Low | Provider-level rate limiting |
| Breaking changes to AI SDK | Low | Medium | Pin versions, test updates |

### 11.3 Rollback Strategy

1. **Feature Flag**: All new code behind feature flag
2. **Parallel Systems**: Old flow works alongside new
3. **Database Compatibility**: Schema unchanged
4. **Gradual Migration**: One handler at a time

---

## 12. Code References

### 12.1 Go Files to Modify

| File | Action | Notes |
|------|--------|-------|
| `main.go` | Add HTTP server startup | Parallel to work queue |
| `cmd/run.go` | Add `--http-port` flag | Configuration |
| `pkg/api/server.go` | **CREATE** | HTTP routes |
| `pkg/api/handlers/*.go` | **CREATE** | Tool handlers |
| `pkg/llm/client.go` | Keep for reference | Don't delete yet |
| `pkg/llm/execute-action.go` | Extract `PerformStringReplacement` | Make public |
| `pkg/listener/start.go` | Remove LLM handlers | After migration |

### 12.2 Next.js Files to Create

| File | Purpose |
|------|---------|
| `app/api/chat/route.ts` | Main AI SDK endpoint |
| `lib/ai/provider.ts` | Provider factory |
| `lib/ai/models.ts` | Model definitions |
| `lib/ai/config.ts` | Configuration |
| `lib/ai/prompts.ts` | System prompts |
| `lib/ai/tools/index.ts` | Tool exports |
| `lib/ai/tools/text-editor.ts` | text_editor tool |
| `lib/ai/tools/subchart.ts` | subchart_version tool |
| `lib/ai/tools/kubernetes.ts` | kubernetes_version tool |
| `lib/ai/tools/validate.ts` | validateChart tool |

### 12.3 Files to Delete (After Migration)

| File | Reason |
|------|--------|
| `lib/llm/prompt-type.ts` | Orphaned, unused |
| `pkg/llm/conversational.go` | Replaced by AI SDK |
| `pkg/llm/initial-plan.go` | Replaced by AI SDK |
| `pkg/llm/plan.go` | Replaced by AI SDK |
| `pkg/llm/summarize.go` | Replaced by AI SDK |
| `pkg/llm/expand.go` | Replaced by AI SDK |
| `pkg/llm/cleanup-converted-values.go` | Replaced by AI SDK |
| `pkg/llm/convert-file.go` | Replaced by AI SDK |

### 12.4 Key Line References

| Reference | Description |
|-----------|-------------|
| `conversational.go:14` | `ConversationalChatMessage()` main function |
| `conversational.go:136-141` | Anthropic streaming setup |
| `conversational.go:99-128` | Tool definitions |
| `conversational.go:170-220` | Tool execution loop |
| `execute-action.go:437` | `ExecuteAction()` entry point |
| `execute-action.go:510-532` | `text_editor` tool definition |
| `execute-action.go:542-673` | Agentic tool execution loop |
| `execute-action.go:238-317` | `PerformStringReplacement()` |
| `system.go:20-55` | `commonSystemPrompt` |
| `system.go:111-120` | `executePlanSystemPrompt` |
| `start.go:15-111` | All work queue handlers |

---

## Conclusion

This research document provides a comprehensive blueprint for **Option A: Full Migration** - moving all LLM orchestration from Go to Next.js with Vercel AI SDK, while preserving Go as a tool execution service.

### Key Benefits

1. **Meets Requirements**: Fulfills Replicated_Chartsmith.md's mandate to "Replace direct Anthropic SDK calls with AI SDK Core's unified API"
2. **Multi-Provider Support**: OpenRouter enables easy provider switching
3. **Simplified Architecture**: Single LLM integration point in Next.js
4. **Preserved Complexity**: Go keeps the hard stuff (fuzzy matching, Helm SDK)
5. **Standard Patterns**: AI SDK's `streamText`, `useChat`, and `tool()` are well-documented

### Recommended Next Steps

1. Create new PR1 Tech PRD based on this research
2. Start with Phase 1: Go HTTP server + AI SDK foundation
3. Implement and test `conversational` chat first
4. Gradually migrate remaining handlers
5. Keep parallel systems until fully validated

---

*Document End - Full Migration Option A Research*
