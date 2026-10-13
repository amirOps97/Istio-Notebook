# 01 · What is Istio?

[⬅ Back to index](README.md)

## The problem
In a microservices system, every service needs the same "plumbing":

- Encrypting traffic between services
- Retries, timeouts, circuit breaking
- Knowing who is calling whom (and blocking who shouldn't)
- Metrics, logs and traces for every call
- Gradual rollouts (canary, A/B)

Implementing this inside every app, in every language, is repetitive and error-prone.

## The idea: a service mesh
A **service mesh** moves that plumbing out of the application and into the
**infrastructure layer**. Network proxies sit next to (or near) each workload and
handle traffic on its behalf. Developers write business logic; the mesh handles networking.

## What Istio gives you

| Area | Features |
|------|----------|
|  Traffic management | Routing rules, traffic splitting, retries, timeouts, fault injection, mirroring |
|  Security | Automatic mutual TLS (mTLS), workload identity, authorization policies, JWT validation |
|  Observability | Golden-signal metrics, distributed tracing, access logs — without code changes |
|  Extensibility | Wasm plugins, EnvoyFilter, external authorization |

## Key terms

- **Mesh** — the set of workloads managed by Istio.
- **Data plane** — the proxies that actually carry traffic.
- **Control plane** — `istiod`, which configures the proxies.
- **Envoy** — the high-performance L7 proxy Istio is built on.
- **Workload** — a pod (or VM) running your application.
- **CRD** — Istio is configured via Kubernetes Custom Resources (`VirtualService`, `AuthorizationPolicy`, ...).

## Istio vs alternatives (quick note)
- **Linkerd** — lighter, simpler, fewer features.
- **Cilium Service Mesh** — eBPF-based, tied to Cilium CNI.
- **Consul** — HashiCorp ecosystem, multi-platform.

Istio is the most feature-rich and widely adopted; the trade-off is more complexity.

---
[Next: Architecture ➡](02-architecture.md)
