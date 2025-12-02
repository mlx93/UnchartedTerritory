# Helm Chart Validation Tools Reference

## Overview

This document outlines the validation tools that will be integrated into Chartsmith's Chart Validation Agent feature. These tools form the "intermediate" validation level specified in our requirements.

---

## Validation Stack

```
┌─────────────────────────────────────────────────────────────┐
│                  VALIDATION PIPELINE                         │
├─────────────────────────────────────────────────────────────┤
│  Level 1: Syntax & Structure (helm lint)                    │
│  ├── Chart.yaml validation                                  │
│  ├── Template syntax checking                               │
│  ├── Values schema validation                               │
│  └── Best practices (icons, versions)                       │
├─────────────────────────────────────────────────────────────┤
│  Level 2: Rendering (helm template)                         │
│  ├── Template rendering with values                         │
│  ├── YAML output generation                                 │
│  └── Conditional logic verification                         │
├─────────────────────────────────────────────────────────────┤
│  Level 3: K8s Best Practices (kube-score)                   │
│  ├── Security context checks                                │
│  ├── Resource limits verification                           │
│  ├── Probe configuration                                    │
│  └── Network policy validation                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Tool 1: helm lint

### Purpose
Examines a chart for possible issues, checking structure and best practices.

### Command
```bash
helm lint [CHART_PATH] [flags]
```

### Key Flags
| Flag | Description |
|------|-------------|
| `--strict` | Fail on lint warnings (not just errors) |
| `--with-subcharts` | Also lint subcharts |
| `--quiet` | Print only warnings and errors |
| `--set` | Set values on command line |
| `--values` | Specify values in YAML file |
| `--kube-version` | Kubernetes version for capability checks |

### Output Types
- `[INFO]` - Recommendations (e.g., missing icon)
- `[WARNING]` - Potential issues
- `[ERROR]` - Chart will fail to install

### Example Output
```
==> Linting ./mychart
[INFO] Chart.yaml: icon is recommended
[ERROR] Chart.yaml: version is required
[WARNING] templates/deployment.yaml: Deprecated API version
Error: 1 chart(s) linted, 1 chart(s) failed
```

### Exit Codes
- `0` - No errors
- `1` - Errors found

---

## Tool 2: helm template

### Purpose
Renders chart templates locally without installing to a cluster.

### Command
```bash
helm template [NAME] [CHART] [flags]
```

### Key Flags
| Flag | Description |
|------|-------------|
| `--debug` | Enable verbose output |
| `--validate` | Validate against Kubernetes schema |
| `--set` | Set values on command line |
| `--values` | Specify values file |
| `--show-only` | Only show specific templates |
| `--output-dir` | Write rendered templates to directory |

### Use Cases
1. **Syntax Verification**: Ensure templates render without errors
2. **Output Inspection**: See what will be deployed
3. **Pipeline Integration**: Feed output to other validators

### Example
```bash
# Render and validate
helm template my-release ./mychart --validate

# Render specific template
helm template my-release ./mychart --show-only templates/deployment.yaml

# Pipe to kube-score
helm template my-release ./mychart | kube-score score -
```

---

## Tool 3: kube-score

### Purpose
Static analysis of Kubernetes YAML for reliability and security recommendations.

### Command
```bash
kube-score score [FILE/STDIN] [flags]
```

### Key Flags
| Flag | Description |
|------|-------------|
| `--output-format` | `human`, `json`, `ci`, or `sarif` |
| `--kubernetes-version` | Target K8s version (default v1.18) |
| `--exit-one-on-warning` | Exit 1 on warnings |
| `--ignore-test` | Disable specific tests |
| `--enable-optional-test` | Enable optional tests |

### Check Categories

#### Critical Checks
- Container resource limits (CPU/memory)
- Pod security context
- Network policies
- Image pull policy

#### Warning Checks
- Readiness/liveness probes
- Image tag best practices (avoid `:latest`)
- Anti-affinity rules
- Service targeting

### Output Format (Human)
```
apps/v1/Deployment my-app
    [CRITICAL] Container Resources
        · Container does not have a CPU limit defined
        · Container does not have a memory limit defined
    [WARNING] Container Image Pull Policy
        · Image pull policy is not set to Always
```

### Output Format (JSON)
```json
{
  "score": 5,
  "scoring": {
    "total": 10,
    "passed": 5,
    "critical": 2,
    "warning": 3
  },
  "checks": [...]
}
```

### Integration with Helm
```bash
helm template my-release ./mychart | kube-score score -
```

---

## Combined Validation Pipeline

### Go Implementation Pseudocode

```go
type ValidationResult struct {
    Tool       string           `json:"tool"`
    Status     string           `json:"status"` // pass, warn, fail
    Issues     []ValidationIssue `json:"issues"`
    RawOutput  string           `json:"raw_output"`
}

type ValidationIssue struct {
    Severity   string `json:"severity"` // info, warning, critical, error
    Message    string `json:"message"`
    File       string `json:"file,omitempty"`
    Line       int    `json:"line,omitempty"`
    Suggestion string `json:"suggestion,omitempty"`
}

func ValidateChart(chartPath string, values map[string]any) (*ValidationReport, error) {
    report := &ValidationReport{}
    
    // Step 1: helm lint
    lintResult, err := runHelmLint(chartPath, values)
    report.LintResult = lintResult
    
    // Step 2: helm template (render)
    renderedYAML, err := runHelmTemplate(chartPath, values)
    if err != nil {
        report.TemplateResult = &ValidationResult{
            Tool:   "helm-template",
            Status: "fail",
            Issues: parseTemplateError(err),
        }
        return report, nil // Return partial results
    }
    report.TemplateResult = &ValidationResult{Tool: "helm-template", Status: "pass"}
    
    // Step 3: kube-score (only if template succeeded)
    scoreResult, err := runKubeScore(renderedYAML)
    report.ScoreResult = scoreResult
    
    return report, nil
}
```

### API Response Structure

```json
{
  "validation": {
    "overall_status": "warning",
    "timestamp": "2024-01-15T10:30:00Z",
    "results": {
      "helm_lint": {
        "status": "pass",
        "issues": []
      },
      "helm_template": {
        "status": "pass",
        "rendered_resources": 5
      },
      "kube_score": {
        "status": "warning",
        "score": 7,
        "issues": [
          {
            "severity": "critical",
            "resource": "Deployment/my-app",
            "check": "container-resources",
            "message": "Container does not have a memory limit defined",
            "suggestion": "Set resources.limits.memory in your container spec"
          }
        ]
      }
    }
  }
}
```

---

## AI Integration Points

### Tool Calling Schema (Vercel AI SDK)

```typescript
const validateChartTool = tool({
  description: 'Validate a Helm chart for syntax, rendering, and best practices',
  inputSchema: z.object({
    chartPath: z.string().describe('Path to the Helm chart directory'),
    values: z.record(z.any()).optional().describe('Values to use for rendering'),
    strictMode: z.boolean().optional().describe('Fail on warnings'),
  }),
  execute: async ({ chartPath, values, strictMode }) => {
    const response = await fetch('/api/validate', {
      method: 'POST',
      body: JSON.stringify({ chartPath, values, strictMode }),
    });
    return response.json();
  },
});
```

### AI Interpretation Flow

```
User: "Validate my chart and fix any issues"
       │
       ▼
┌──────────────────────────────────────┐
│  AI calls validateChartTool          │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  Go backend runs validation pipeline │
│  - helm lint                         │
│  - helm template                     │
│  - kube-score                        │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  Results returned to AI              │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  AI interprets results, suggests     │
│  fixes in natural language           │
└──────────────────────────────────────┘
       │
       ▼
User sees: "I found 3 issues with your chart:
1. [CRITICAL] Missing memory limits on deployment...
   Fix: Add resources.limits.memory: '256Mi' to your container spec
2. [WARNING] Image uses :latest tag...
   Fix: Pin to specific version like nginx:1.21.0
3. [INFO] Missing icon in Chart.yaml...
   This is optional but recommended for Helm Hub visibility"
```

---

## Installation Requirements

### Go Backend Dependencies

```bash
# Helm CLI (required)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# kube-score (required for best practices checks)
# Option 1: Binary
wget https://github.com/zegl/kube-score/releases/download/v1.18.0/kube-score_1.18.0_linux_amd64.tar.gz

# Option 2: Go install
go install github.com/zegl/kube-score/cmd/kube-score@latest

# Option 3: Docker (includes Helm)
docker pull zegl/kube-score:v1.18.0
```

### Version Compatibility
| Tool | Minimum Version | Notes |
|------|-----------------|-------|
| Helm | 3.0+ | Helm 2 deprecated |
| kube-score | 1.16+ | Multiarch images available |
| Go | 1.21+ | For kube-score build |

---

## References

- [helm lint documentation](https://helm.sh/docs/helm/helm_lint/)
- [helm template documentation](https://helm.sh/docs/helm/helm_template/)
- [kube-score GitHub](https://github.com/zegl/kube-score)
- [kube-score checks list](https://kube-score.com/)
