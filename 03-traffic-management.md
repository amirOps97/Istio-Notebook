# 03 · Traffic Management

## The core resources

| Resource | Answers the question |
|----------|----------------------|
| **Gateway** | Which traffic enters/leaves the mesh, on which ports/hosts? |
| **VirtualService** | *Where* does a request go? (routing rules) |
| **DestinationRule** | *How* is traffic handled once it gets there? (subsets, LB, TLS, circuit breaking) |
| **ServiceEntry** | Adds external services to the mesh registry |
| **Sidecar** | Limits what config/hosts a proxy sees (performance) |

> Mental model: **VirtualService = routing**, **DestinationRule = policies after routing**.



## VirtualService + DestinationRule (Canary / weighted routing)

```yaml
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: myapp  # "just" a name for this virtmyappmespace: default
spec:
  myapp-service.default.svc.cluster.local  # a regular K8S service name that we're applying routing rules to.
  http:  # defining routing rules
    - route:
        - destination: myapp.default.svc.cluster.local # The Target K8S service
            subset: safe-group  # The name defined in the DestinationRule
          weight: 90
        - destination: myapp.default.svc.cluster.local # The Target K8S service
            subset: canary-group  # The name defined in the DestinationRule
          weight: 10
---
kind: DestinationRule       # Defining which pods should be part of each subset
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: myapp # This can be anythingmyappmespace: default
spec:
  host: myapp-service # Service
  subsets:
    - labels:   # SELECTOR
        version: safe # find pods with label "safe"
      name: safe-group
    - labels:
        version: risky 
      name: canary-group
```

## Gateway + VirtualService (ingress) #1

```yaml

apiVersion: networking.istio.io/v1alpha3
kind: Gateway # instrcut the istio ingress gateway to listen to http traffic
metadata:
  name: ingress-gateway-configuration
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"   # Domain name of the external website
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  nmyappmespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "*"
  gateways:
    - ingress-gateway-configuration
  http:
    - match:
      - uri:  # IF
          prefix: "/experimental"
      - uri:  # OR
          prefix: "/canary"
      route: # THEN
      - destination:
          host: myapp-service
          subset: experimental
    - match:
      - uri :
          prefix: "/"
      route:
      - destination:
          host:myapp-service
          subset: original
---
kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: myamyappmespace: default
spec:
  host: myapp-service
  subsets:
    - labels:
        version: original
      name: original
    - labels:
        version: experimental
      name: experimental
```

## Gateway + VirtualService (ingress) #2

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway 
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port: { number: 80, name: http, protocol: HTTP }
      hosts: ["myapp.example.com"] # Domain name of the external website
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp  # "just" a name for this virtualservice
spec:
  hosts: ["myapp.example.com"]  # which incoming host are we applying the proxy rules to???
  gateways: ["my-gateway"]
  http:
    - match:
        - uri: { prefix: /api }
      route:
        - destination: { host: api-service, port: { number: 8080 } }
    - route:
        - destination: { host: frontend }
```


## Gateway + VirtualService (ingress) #3

```yaml

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"   # Domain name of the external website
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  nmyappmespace: default
spec:
  hosts:      
    - "*" 
  gateways:
    - ingress-gateway-configuration
  http:
    - route:
        - destination:
            host: myapp-service
            subset: original
          weight: 90
        - destination:
            host: myapp-service
            subset: experimental
          weight: 10
---
kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  nmyappmespace: default
spec:
  host: myapp-service
  subsets:
    - labels:
        version: original
      name: original
    - labels:
        version: experimental
      name: experimental
```

## Gateway + VirtualService (ingress) #4

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*.example.com"
    - "example.com"

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-servmyappmespace: default
spec:
  hosts:
  - "myapp-service"
  gateways:
  - my-gateway
  http:
  - match:
    - headers:
        my-header:    # IF my-header: expermental Headr has been set (catch a CONDITION), ! IMPORTANT: header must be propagated in application code meaning when the service calls another service, it MUST copy the header.
          exact: experimental
    route:            # THEN,
    - destination:    
        host: myapp-service
        subset: experimental-group
  - route:            # ELSE (catch ALL),
    - destination:
        host: myapp-service
        subset: original-group
      
---
kind: DestinationRule      
apiVersion: networking.istio.io/v1alpha3
metadata:
  namyappmespace: default
spec:
  host: myapp-service 
  subsets:
    - labels:  
        version: original
      name: original-group
    - labels:
        version: experimental
      name: experimental-group
```


## Gateway + VirtualService (ingress) - subdomain routing
```yaml

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*.example.com"   # Domain name of the external website
    - "example.com"
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: myapp
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "example.com"
  gateways:
    - ingress-gateway-configuration
  http:
    - route:
      - destination:
          host: myapp-service
          subset: original
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: experimental-myapp
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "experimental.fleetman.com"
  gateways:
    - ingress-gateway-configuration
  http:
      - route:
        - destination:
            host: myapp-service
            subset: experimental
---
kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: myapp
  namespace: default
spec:
  host: myapp-service
  subsets:
    - labels:
        version: original
      name: original
    - labels:
        version: experimental
      name: experimental

```

## VirtualService 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-sermyappmespace: default
spec:
  hosts:
  - "myapp-service.default.svc.cluster.local"
  http:
  - match:     # IF my-header: expermental Headr has been set (catch a CONDITION),
    - headers:
        x-header: 
          exact: experimental
    route:    # THEN,
    - destination:
        host: myapp-service.default.svc.cluster.local
        subset: risky-group
  - route:    # ELSE (catch ALL)
    - destination:
        host: myapp-service.default.svc.cluster.local
        subset: safe-group

---
kind: DestinationRule      
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: my-app
  namespace: default
spec:
  host: myapp-service 
  subsets:
    - labels:  
        version: safe
      name: safe-group
    - labels:
        version: risky
      name: risky-group

```

## Timeouts & retries

```yaml
  http:
    - route:
        - destination: { host: ratings }
      timeout: 5s
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,connect-failure,reset
```

## Fault injection (chaos testing)

```yaml
  http:
    - fault:
        delay:
          percentage: { value: 10 }
          fixedDelay: 5s
        abort:
          percentage: { value: 5 }
          httpStatus: 500
      route:
        - destination: { host: ratings }
```

## Circuit breaking & outlier detection

to prevent cascading failure

```yaml

apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp: { maxConnections: 100 }
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50
    loadBalancer:
      simple: LEAST_REQUEST
```

## Traffic mirroring (shadowing)

```yaml
  http:
    - route:
        - destination: { host: myapp, subset: v1 }
      mirror: { host: myapp, subset: v2 }
      mirrorPercentage: { value: 100 }
```
Mirrored responses are discarded — safe for testing a new version with real traffic.

## External services: ServiceEntry

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts: ["api.github.com"]
  ports:
    - { number: 443, name: https, protocol: TLS }
  resolution: DNS
  location: MESH_EXTERNAL
```

Lock down egress so only registered services are reachable:
```yaml
meshConfig:
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
```

## Kubernetes Gateway API (the newer way)
Istio supports the standard Gateway API (`Gateway`, `HTTPRoute`) and it's the
recommended direction, especially in ambient mode.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      port: 80
      protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
spec:
  parentRefs: [{ name: my-gateway }]
  rules:
    - backendRefs:
        - { name: reviews-v1, port: 9080, weight: 90 }
        - { name: reviews-v2, port: 9080, weight: 10 }
```

---
