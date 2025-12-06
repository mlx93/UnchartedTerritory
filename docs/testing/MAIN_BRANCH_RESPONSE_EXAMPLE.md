# Main Branch Response Example

**Prompt:** "create a simple nginx deployment"

## What Main Branch Shows

In the main branch (Go worker flow), after sending a message, you get:

1. **Proposed Plan (review)** - A detailed plan document
2. **"Create Chart" button** - User must approve before execution

### Example Response

```
ChartSmith
Rev #0

Proposed Plan
(review)

Plan for Creating an Nginx Deployment Helm Chart

I will create a comprehensive Helm chart that deploys a production-ready nginx web server to Kubernetes clusters. This chart will be designed to support Kubernetes versions 1.20 through 1.30 and Helm versions 3.8 and above, ensuring compatibility across a wide range of modern Kubernetes environments including managed services like EKS, GKE, AKS, as well as on-premises distributions such as OpenShift, RKE2, and vanilla Kubernetes.

The chart will establish nginx as the primary application workload, utilizing the official nginx container image from Docker Hub as the default. The deployment architecture will center around a standard Kubernetes Deployment resource that manages nginx pod replicas, with configurable replica counts to support both development and production scenarios. End customers will be able to customize the nginx image repository, tag, and pull policy to accommodate their specific requirements, including the use of private registries or custom nginx builds.

Service exposure will be handled through a Kubernetes Service resource with configurable service types including ClusterIP for internal access, NodePort for direct node access, and LoadBalancer for cloud environments. The chart will include comprehensive ingress support with configurable ingress classes, annotations for various ingress controllers including NGINX Ingress Controller, Traefik, and cloud-native solutions, and support for TLS termination with certificate management.

Resource management will be fully parameterized, allowing end customers to specify CPU and memory requests and limits appropriate for their workload requirements. The chart will include horizontal pod autoscaling capabilities with configurable metrics including CPU and memory utilization thresholds, enabling automatic scaling based on demand.

Security considerations will be thoroughly addressed through configurable pod security contexts, support for running as non-root users, and integration with Kubernetes service accounts including support for workload identity in cloud environments. The chart will provide options for security context constraints to ensure compatibility with security-focused distributions like OpenShift.

Configuration flexibility will be achieved through support for custom nginx configurations via ConfigMaps, persistent volume claims for serving static content, and environment variable injection. Health checking will be implemented through configurable liveness and readiness probes with HTTP endpoints, ensuring proper pod lifecycle management.

The chart will incorporate the replicated dependency as specified, maintaining version 1.0.0-beta.32 from the OCI registry at registry.replicated.com/library. This dependency will be properly integrated to support enterprise deployment scenarios and application lifecycle management requirements.

Operational features will include comprehensive labeling following Kubernetes recommended practices, support for pod annotations and labels for integration with monitoring and logging systems, node selection capabilities through node selectors and affinity rules, and toleration support for specialized node configurations.

The values configuration will be structured to provide sensible defaults while maintaining extensive customization options. Default resource allocations will be conservative to ensure the chart can run in resource-constrained environments while providing clear guidance for production scaling. The chart will include detailed documentation within the values file explaining each configuration option and its impact on the deployment.

Testing and validation will be designed to work across multiple Kubernetes distributions, with particular attention to namespace restrictions, security policy compliance, and networking model variations. The chart structure will follow Helm best practices for template organization, helper function usage, and conditional resource creation.

[Create Chart] button at bottom
```

---

## Key Difference: Main vs AI SDK Branch

| Aspect | Main Branch (Go Worker) | AI SDK Branch |
|--------|------------------------|---------------|
| **Flow** | Plan → Review → Approve → Execute | Direct streaming response |
| **UI** | Shows "Proposed Plan (review)" with Create Chart button | Streams assistant text directly |
| **User Action** | Must click "Create Chart" to proceed | No approval step needed |
| **Real-time** | Updates after refresh | Should stream live |

## Why AI SDK Branch Behaves Differently

The `myles/vercel-ai-sdk-migration` branch uses the Vercel AI SDK which:
1. Streams responses directly to the chat UI
2. Doesn't use the Go worker's plan/review workflow
3. Has `NEXT_PUBLIC_USE_AI_SDK_CHAT=true` feature flag

To achieve parity, the AI SDK path would need to:
1. Either integrate with the Go plan system, OR
2. Implement its own plan/review UI flow, OR
3. Accept the simpler streaming-only experience

---

*Captured: Dec 5, 2025*

