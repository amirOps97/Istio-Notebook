# 03 · Security


## Identity
Every workload gets a cryptographic identity based on its **Kubernetes service account**,
in SPIFFE format:

```
spiffe://cluster.local/ns/<namespace>/sa/<service-account>
```

`istiod` acts as a CA, issues short-lived certificates, and rotates them automatically.

`ServiceAccount:` Provides an identity for a workload in Kubernetes, such as frontend or payment. A Pod uses the ServiceAccount assigned to it.

`istiod` Uses SPIFFE-based workload identities in certificates to authenticate services when they communicate over mTLS. Istio's AuthorizationPolicy lets administrators control which workloads can access a service and, for supported HTTP traffic, which methods or paths they can access.

`SPIFFE:` Defines a standard format for identifying workloads. For example:
```
spiffe://cluster.local/ns/default/sa/frontend
````

| Part | Meaning |
|------|-----------|
| `cluster.local` | the cluster(trust domain) |
| `ns/default` | namespace default |
| `sa/frontend` | service accoiunt frontend |



## Mutual TLS (mTLS)
Both sides present certificates → traffic is encrypted **and** both identities are verified.

### PeerAuthentication modes

| Mode | Behaviour |
|------|-----------|
| `PERMISSIVE` (default) | Accepts both mTLS and plaintext — good for migration |
| `STRICT` | mTLS only — plaintext rejected |
| `DISABLE` | No mTLS |

Mesh-wide strict mTLS (apply in the root namespace):
```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

Per-namespace or per-workload overrides are possible — the most specific one wins.

## Authorization: AuthorizationPolicy

Actions: `ALLOW`, `DENY`, `CUSTOM` (external authz), `AUDIT`.

**Evaluation order:** CUSTOM → DENY → ALLOW. If any ALLOW policy exists for a workload,
requests not matching an ALLOW are denied.


> ALLOW is a guest list (only listed people get in). DENY is a blacklist (everyone gets in except the listed people).


### Deny-all baseline (zero trust)
```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: prod
spec: {}

```


| Field | Missing means |
|------|-----------|
| `selector` | applies to all pods in the namespace prod |
| `action` | defaults to ALLOW (not “no action”) |
| `rules` | no rules, so it matches nothing |

So this is actually an ALLOW policy that allows nothing

> Once a workload has at least one ALLOW policy, any request that doesn’t match an ALLOW is denied.

Every pod in prod now has an ALLOW policy (because there’s no selector).
That policy has no rules, so no request can ever match it.
So every request is denied


### Allow a specific service to call another
```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy # Istio Resource for allowing or denying requests
metadata:
  name: allow-frontend-to-api # just a label for this rule - kubectl get authorizationpolicy
  namespace: prod # only affects pods in prod
spec:
  selector: # without the selector, the rule appies to every pod in the namespace
    matchLabels: { app: api } # who is protected - which pods the guard stands in front of - receiving side
  action: ALLOW # if a request matches the rule below, let it in
  rules: # a rule has two questions: who is calling (from) and what are they trying to do (to)
    - from:
        - source: # who is calling
            principals: ["cluster.local/ns/prod/sa/frontend"] # identity of the caller (SPIFFE format)
      to: # what are they allowed to do
        - operation:
            methods: ["GET", "POST"] # DELETE or PUT would not match
            paths: ["/api/*"] # only URLs starting with /api/. A call to /admin would not match
```
>  Once a workload has at least one ALLOW policy, anything that doesn’t match an ALLOW is automatically denied

### Deny by namespace
```yaml
spec:
  action: DENY
  rules:
    - from:
        - source:
            namespaces: ["untrusted"] # only Requests coming from pods in the untrusted namespace: 403 RBAC: access denied
```

> If mTLS is in PERMISSIVE mode and a pod in untrusted has no sidecar, it sends plaintext with no certificate. Then Istio sees no namespace at all, the DENY rule doesn’t match, and the request is allowed. So a DENY based on identity can be bypassed by simply not having an identity


> In **ambient mode**, L4 rules (principals, namespaces, ports) are enforced by ztunnel.
> L7 rules (paths, methods, headers) require a **waypoint proxy**.

## End-user auth: RequestAuthentication (JWT)

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication  # is there Authorization: Bearer ... header? if yes:is the signature valid (checked with the keys from jwksUri)? Is the issuer correct? Is the token not expired? If everything is valid, it extracts the user identity, called the request principal:    https://auth.example.com/alice       (issuer + "/" + sub)
metadata:
  name: jwt-auth
  namespace: prod
spec:
  selector:
    matchLabels: { app: api }
  jwtRules:
    - issuer: "https://auth.example.com" # the login server we trust. The iss field inside the token must equal exactly https://auth.example.com. A token from any other issuer is rejected.
      jwksUri: "https://auth.example.com/.well-known/jwks.json" # the address where the login server publishes its public keys.The sidecar downloads these keys and uses them to check the signature on each token
```


> RequestAuthentication only says “if you show an ID card, it must be real.” It does not say “you must show an ID card.” So someone who sends no token at all walks right in.


RequestAuthentication only **rejects invalid** tokens — requests with **no** token still pass.
To require a token, pair it with an AuthorizationPolicy:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: prod
spec:
  selector:
    matchLabels: { app: api }
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["*"] # the request must have a request principal, meaning a user identity that came from a valid JWT. "*" means “any user, I don’t care who, as long as there is one
```

## TLS at the ingress gateway
```yaml
  servers:
    - port: { number: 443, name: https, protocol: HTTPS }
      tls:
        mode: SIMPLE
        credentialName: myapp-tls-secret   # k8s secret in the gateway's namespace
      hosts: ["myapp.example.com"]
```

---

