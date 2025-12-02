# PR2 Supplemental Implementation Details

This document contains component pseudocode and rendering logic extracted for reference during implementation. Primary specs are in PRD_Product_PR2.md and PRD_Tech_PR2.md.

---

## ValidationResults Component

**File**: `src/components/chat/ValidationResults.tsx`

**Props**:
- `result`: ValidationResult object from Go backend

**Render Logic**:
```
COMPONENT ValidationResults(result):
  RENDER container with header showing overall status
  
  // Status badge with color
  statusColor = SWITCH result.overall_status:
    "pass" -> green
    "warning" -> yellow  
    "fail" -> red
  
  RENDER StatusBadge(result.overall_status, statusColor)
  
  // Summary stats
  IF result.results.kube_score exists:
    RENDER "Score: {score}/10 ({passed}/{total} checks)"
  
  // Issues list grouped by severity
  criticalIssues = filter(allIssues, severity === "critical")
  warningIssues = filter(allIssues, severity === "warning")
  infoIssues = filter(allIssues, severity === "info")
  
  IF criticalIssues.length > 0:
    RENDER IssueSection("Critical", criticalIssues, "red")
  
  IF warningIssues.length > 0:
    RENDER IssueSection("Warnings", warningIssues, "yellow")
  
  IF infoIssues.length > 0:
    RENDER IssueSection("Info", infoIssues, "blue")
  
  IF allIssues.length === 0:
    RENDER "✅ No issues found"
```

**IssueItem Subcomponent**:
```
COMPONENT IssueItem(issue):
  RENDER row with:
    - Severity icon (critical=🔴, warning=🟡, info=🔵)
    - Source badge (lint/template/kube-score)
    - Message text
    - File:line reference if available
    - Expandable suggestion section
```

---

## LiveProviderSwitcher Component

**File**: `src/components/chat/LiveProviderSwitcher.tsx`

**Props**:
- `currentProvider`: Currently selected provider config
- `onSwitch`: Callback when provider changes
- `disabled`: Optional, disable during streaming

**State**:
- `isOpen`: Dropdown visibility

**Render Logic**:
```
COMPONENT LiveProviderSwitcher(currentProvider, onSwitch, disabled):
  STATE isOpen = false
  
  RENDER button showing current provider:
    onClick = toggle isOpen (unless disabled)
    display = currentProvider.name + dropdown arrow
    
  IF isOpen:
    RENDER dropdown overlay:
      FOR each provider in AVAILABLE_PROVIDERS:
        RENDER option row:
          - Provider name
          - Checkmark if current
          - onClick = onSwitch(provider), close dropdown
      
      // Click outside to close
      onClickOutside = () => setIsOpen(false)
```

**Provider Switch Handler** (in parent Chat component):
```
FUNCTION handleProviderSwitch(newProvider):
  // Just update the provider state
  // useChat body will automatically include new provider on next message
  setSelectedProvider(newProvider)
  
  // Optional: Show brief toast "Switched to {name}"
  showToast(`Now using ${newProvider.name}`)
```

---

## Tool Result Rendering in Message Component

**File**: Modify existing message component

**Part Type Handling**:
```
FUNCTION renderMessagePart(part):
  SWITCH part.type:
    CASE "text":
      RETURN <MarkdownContent text={part.text} />
    
    CASE "tool-call":
      IF part.toolName === "validateChart":
        RETURN <ValidationInProgress chartPath={part.args.chartPath} />
      RETURN <GenericToolCall toolName={part.toolName} args={part.args} />
    
    CASE "tool-result":
      IF part.toolName === "validateChart":
        RETURN <ValidationResults result={part.result} />
      RETURN <GenericToolResult toolName={part.toolName} result={part.result} />
    
    DEFAULT:
      RETURN null
```

**ValidationInProgress Component**:
```
COMPONENT ValidationInProgress(chartPath):
  RENDER card with:
    - Spinner icon
    - "Validating chart..." text
    - Chart path being validated
    - Optional: stage indicators (lint ✓, template ⏳, kube-score ○)
```

---

## Chat Component Integration

**Updated Chat component structure**:
```
COMPONENT Chat:
  STATE selectedProvider = DEFAULT_PROVIDER
  
  // useChat from PR1, now with tool support
  const { messages, input, handleSubmit, ... } = useChat({
    api: "/api/chat",
    body: {
      provider: selectedProvider.id,
      model: selectedProvider.model
    }
  })
  
  RENDER:
    <ChatHeader>
      <Title>Chartsmith Chat</Title>
      <LiveProviderSwitcher
        currentProvider={selectedProvider}
        onSwitch={setSelectedProvider}
        disabled={status === "streaming"}
      />
    </ChatHeader>
    
    <MessageList>
      FOR each message in messages:
        <Message>
          FOR each part in message.parts:
            {renderMessagePart(part)}
        </Message>
    </MessageList>
    
    <ChatInput 
      value={input}
      onChange={handleInputChange}
      onSubmit={handleSubmit}
      disabled={status !== "ready"}
    />
```

---

## validateChart Tool Definition

**File**: `src/lib/tools/validateChart.ts`

```
IMPORT { tool } from 'ai'
IMPORT { z } from 'zod'

EXPORT validateChart = tool({
  description: "Validate a Helm chart for syntax errors, rendering issues, 
                and Kubernetes best practices. Use when user asks to validate, 
                check, lint, or verify their chart.",
  
  inputSchema: z.object({
    chartPath: z.string().describe("Path to chart directory"),
    values: z.record(z.any()).optional().describe("Values overrides"),
    strictMode: z.boolean().optional().describe("Fail on warnings"),
    kubeVersion: z.string().optional().describe("Target K8s version")
  }),
  
  execute: ASYNC (input) => {
    response = await fetch(`${GO_BACKEND_URL}/api/validate`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(input)
    })
    
    IF not response.ok:
      THROW new Error(`Validation failed: ${response.statusText}`)
    
    RETURN response.json()
  }
})
```

**Register in streamText**:
```
// In /api/chat/route.ts
import { validateChart } from '@/lib/tools/validateChart'

const result = await streamText({
  model: getModel(provider),
  messages,
  tools: {
    validateChart
  }
})
```

---

## Grade to Severity Mapping (kube-score)

```
FUNCTION mapGradeToSeverity(grade: number) -> string:
  IF grade <= 1:
    RETURN "critical"
  ELSE IF grade <= 5:
    RETURN "warning"
  ELSE:
    RETURN "info"  // grade 10 = pass, not shown as issue
```

## Suggestion Lookup Table

```
SUGGESTIONS = {
  "container-resources": "Add resources.limits.memory and resources.limits.cpu",
  "container-security-context": "Set securityContext.runAsNonRoot: true",
  "container-image-tag": "Use specific image tag instead of :latest",
  "pod-probes": "Add readinessProbe and livenessProbe",
  "pod-networkpolicy": "Create a NetworkPolicy for this namespace",
  "deployment-replicas": "Set replicas >= 2 for high availability"
}

FUNCTION generateSuggestion(checkName: string) -> string:
  RETURN SUGGESTIONS[checkName] OR "Review Kubernetes best practices documentation"
```
