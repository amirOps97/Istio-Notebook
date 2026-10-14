# 06 · Troubleshooting

## Start here
```bash
istioctl analyze --all-namespaces      # finds config mistakes
istioctl proxy-status                  # is every proxy SYNCED with istiod?
kubectl get pods -n istio-system       # is the control plane healthy?
```

## Inspect a proxy's actual config
```bash
istioctl proxy-config listeners <pod> -n <ns>
istioctl proxy-config routes    <pod> -n <ns>
istioctl proxy-config clusters  <pod> -n <ns>
istioctl proxy-config endpoints <pod> -n <ns>
istioctl proxy-config secret    <pod> -n <ns>   # certificates

# What policies apply to this pod?
istioctl x describe pod <pod> -n <ns>
```

## Logs
```bash
kubectl logs <pod> -c istio-proxy -n <ns>          # sidecar logs
kubectl logs -n istio-system deploy/istiod         # control plane
kubectl logs -n istio-system ds/ztunnel            # ambient L4

# Raise proxy log level temporarily
istioctl proxy-config log <pod> -n <ns> --level debug
```

## Common problems

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Pod shows `1/1`, no sidecar | Namespace not labeled, or pod created before labeling | Label namespace, `kubectl rollout restart` |
| `503 UF` / `upstream connect error` | Upstream unreachable, wrong port, mTLS mismatch | Check endpoints, port names, PeerAuthentication |
| `503 NR` (no route) | VirtualService host/gateway mismatch | `istioctl analyze`, check `hosts` and `gateways` |
| `403 RBAC: access denied` | AuthorizationPolicy blocking | `istioctl x describe pod`, review ALLOW rules |
| Subset not found | VirtualService references subset with no DestinationRule | Create DestinationRule with that subset |
| Protocol issues | Service port not named/typed | Name ports `http-*`, `grpc-*`, `tcp-*` or set `appProtocol` |
| Traces broken | App not forwarding trace headers | Propagate headers in app code |
| App can't reach anything at startup | App starts before sidecar is ready | Set `holdApplicationUntilProxyStarts: true` |
| Job/CronJob never completes | Sidecar keeps running | Use native sidecars (k8s ≥1.28) or ambient mode |

## Envoy response flags (in access logs)

| Flag | Meaning |
|------|---------|
| `UF` | Upstream connection failure |
| `UH` | No healthy upstream |
| `NR` | No route configured |
| `URX` | Retry limit reached |
| `UO` | Upstream overflow (circuit breaker) |
| `DC` | Downstream connection terminated |
| `RL` | Rate limited |

## Debugging checklist
1. `istioctl analyze` — any warnings?
2. `istioctl proxy-status` — all SYNCED?
3. Does the pod have a sidecar / is the namespace in ambient?
4. Are Service ports named correctly?
5. Does the VirtualService `host` match the Service name?
6. Is there a DestinationRule for every subset referenced?
7. mTLS mode consistent between client and server?
8. Any AuthorizationPolicy denying the request?
9. Check access logs for the response flag.

---

