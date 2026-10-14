# 05 · Observability

Istio's proxies see every request, so you get telemetry **without changing application code**.

## The three pillars

| Pillar | What Istio provides | Common tool |
|--------|---------------------|-------------|
| Metrics | Request count, duration, size, TCP bytes | Prometheus + Grafana |
| Traces | Spans for each hop | Jaeger, Zipkin, Tempo, OpenTelemetry |
| Logs | Envoy access logs | stdout → Loki / ELK |

Plus **Kiali** for a live service graph of the mesh.

## Install the addons (learning setup)
```bash
kubectl apply -f samples/addons/
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard prometheus
istioctl dashboard jaeger
```

## Key metrics

| Metric | Meaning |
|--------|---------|
| `istio_requests_total` | Request count (labels: source, destination, response_code...) |
| `istio_request_duration_milliseconds` | Latency histogram |
| `istio_request_bytes` / `istio_response_bytes` | Payload sizes |
| `istio_tcp_sent_bytes_total` | TCP traffic |

### Useful PromQL
```promql
# Request rate per service
sum(rate(istio_requests_total{reporter="destination"}[5m])) by (destination_service)

# Error rate (5xx)
sum(rate(istio_requests_total{response_code=~"5.."}[5m]))
  / sum(rate(istio_requests_total[5m]))

# P99 latency
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (le, destination_service))
```

## Distributed tracing — important gotcha 
Envoy creates spans automatically, **but your app must forward trace headers**
from incoming to outgoing requests, otherwise traces break into pieces.

Headers to propagate: `traceparent`, `tracestate` (W3C), `x-request-id`,
and the B3 headers (`x-b3-traceid`, `x-b3-spanid`, `x-b3-parentspanid`, `x-b3-sampled`) if used.

## Telemetry API
Configure telemetry per mesh / namespace / workload.

Enable access logs mesh-wide:
```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
    - providers:
        - name: envoy
```

Set trace sampling:
```yaml
spec:
  tracing:
    - randomSamplingPercentage: 10.0
```

## Kiali
- Visual graph of services and traffic flow
- Shows health, error rates, mTLS status (lock icon)
- Validates Istio config and flags mistakes

---

