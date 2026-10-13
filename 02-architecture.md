# 02 · Architecture


Istio has two halves: a **control plane** that decides, and a **data plane** that does.

## Control plane: `istiod`
A single binary that combines what used to be separate components:

| Responsibility | What it does |
|----------------|--------------|
| **Pilot** (config) | Watches Kubernetes + Istio CRDs, translates them to Envoy config, pushes via xDS |
| **Citadel** (security) | Acts as a CA, issues certificates for workload identity (SPIFFE) |
| **Galley** (validation) | Validates Istio configuration |

## Data plane: two modes

### 1. Sidecar mode (classic)
An Envoy proxy container is injected into every pod. All inbound/outbound traffic is
redirected through it (via iptables set up by an init container or the Istio CNI plugin).

Full L7 features per pod · Extra CPU/memory per pod, pod restart needed to inject/upgrade

### 2. Ambient mode (sidecar-less)
Splits the data plane into two layers:

- **ztunnel** — a per-node L4 proxy (DaemonSet). Handles mTLS, identity, L4 auth and telemetry.
- **waypoint proxy** — an optional, per-namespace (or per-service) Envoy deployment for L7 features (HTTP routing, L7 auth policies, retries).

No sidecars, lower resource use, no pod restarts · L7 features need a waypoint

### Sidecar vs Ambient

| | Sidecar | Ambient |
|--|---------|---------|
| Proxy location | In each pod | ztunnel per node (+ waypoint for L7) |
| Enable for namespace | `istio-injection=enabled` | `istio.io/dataplane-mode=ambient` |
| Resource overhead | Higher | Lower |
| L7 features | Always available | Only via waypoint |
| App restart to join | Yes | No |

## Ingress & Egress gateways
Standalone Envoy deployments at the **edge** of the mesh:
- **Ingress gateway** — traffic entering the mesh from outside.
- **Egress gateway** — controlled exit point for traffic leaving the mesh.

## How config flows
1. You `kubectl apply` an Istio CRD (e.g. `VirtualService`).
2. `istiod` watches it and builds Envoy configuration.
3. `istiod` pushes config to proxies over **xDS** (gRPC streaming).
4. Envoy applies it — no restart needed.

---

### How sidecar injection works: the Mutating Admission Webhook

Istio doesn't add the sidecar to your Deployment YAML. Kubernetes adds it at the moment
the pod is created, using a **Mutating Admission Webhook** served by `istiod`.

#### The flow

1. You create a pod (directly, or through a Deployment/ReplicaSet).
2. The request reaches the **Kubernetes API server**.
3. During admission, the API server checks the `MutatingWebhookConfiguration`
   named `istio-sidecar-injector`.
4. If the pod's **namespace has the right label**, the API server sends an
   `AdmissionReview` request to the istiod webhook (`/inject` endpoint).
5. **istiod mutates the pod spec** and returns a JSON patch that adds:
   - the `istio-proxy` container (Envoy)
   - the `istio-init` init container that sets up iptables rules to redirect traffic
     (not needed when the Istio CNI plugin is used)
   - volumes for certificates and config
6. The API server applies the patch and stores the **modified pod**.
7. The pod starts with **2 containers** (`2/2`): your app and `istio-proxy`.

#### The namespace label qualifies the pod

The webhook only fires for pods in namespaces that have the injection label:

```bash
kubectl label namespace default istio-injection=enabled

# check
kubectl get namespace -L istio-injection
```

When using **revision-based installs** (e.g. canary upgrades of Istio), use the revision label instead:

```bash
kubectl label namespace default istio.io/rev=<revision-name>
```

>  Don't set both `istio-injection` and `istio.io/rev` on the same namespace.
> `istio-injection` takes precedence.


#### Key point: injection happens only at creation time

A webhook only acts on **create** requests. Existing pods are not changed when you label
the namespace, so you need to recreate them:

```bash
kubectl rollout restart deployment -n default
```

#### Inspect the webhook

```bash
kubectl get mutatingwebhookconfiguration
kubectl get mutatingwebhookconfiguration istio-sidecar-injector -o yaml

# check if a pod would be injected, and why
istioctl x check-inject -n default <pod-name>
```