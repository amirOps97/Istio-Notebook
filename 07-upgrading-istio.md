# 07 · Canary Upgrade of Istio


## Why canary instead of in-place?

| | In-place upgrade | Canary (revision-based) upgrade |
|--|------------------|----------------------------------|
| How | Replace istiod with the new version | Run **old and new istiod side by side** |
| Risk | Every workload is affected at once | Move namespaces **one at a time** |
| Rollback | Hard: reinstall the old version | Easy: point the namespace back to the old revision and restart |
| Use for | Labs, small clusters | Production |

## The key idea: revisions

Each control plane installation gets a **revision name**. Workloads choose which control plane they belong to with a **namespace label**:

```
istio.io/rev=<revision>
```

When a pod is created, the API server sends it to the matching istiod's injection webhook, which injects a sidecar of **that** version.



>  Revision names can't contain dots. Use `1-31-1`, not `1.31.1`.


### Injection

When a pod is **created**, Kubernetes asks mutating webhooks whether they want to modify it. Istio's webhook adds the Envoy sidecar. The sidecar version is the version of the istiod whose webhook handled the pod.

> Injection only happens at pod creation. Changing labels or tags does nothing to running pods: they must be restarted.


### Labels

Namespaces choose their istiod with a label:

| Namespace label | Handled by |
|---|---|
| `istio.io/rev=<revision>` | that specific revision |
| `istio-injection=enabled` | whichever revision is the **default** |
| `istio.io/rev=default` | same as `istio-injection=enabled` |

> If a namespace has both `istio-injection` and `istio.io/rev`, `istio-injection` wins and `istio.io/rev` is ignored. Use only one.


### Tags
A tag is a **nickname that points to a revision**. `default` is a special tag used by `istio-injection=enabled`. You can also create your own tags (for example `prod-stable`) and point them at any revision.


## Steps

Here’s the canary path from a non-revisioned 1.30.5 install to 1.31.1. The idea is to run the new control plane (istiod-1-31-1) next to the old one. You then move workloads over one namespace at a time, verify, and only remove the old control plane at the end.

## Step 0: Pre-flight checks

```bash
istioctl version --remote=false     # 1.31.1
istioctl x precheck                  # checks the cluster is ready for 1.31
istioctl analyze -A                  # fix any warnings before you start
```

### Step 1: Install the 1.31 control plane as a revision (without gateways)

To keep step 1 limited to the control plane, use an IstioOperator file that disables gateways for now

```yaml

apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  revision: 1-31-1
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: false
    egressGateways:
    - name: istio-egressgateway
      enabled: false
````

```bash
istioctl install -f canary-1-31-1.yaml -y
```
## Step 2: Verify both control planes are running

```bash
kubectl get pods -n istio-system -l app=istiod --show-labels
# istiod-xxxx          istio.io/rev=default
# istiod-1-31-1-xxxx   istio.io/rev=1-31-1

kubectl get mutatingwebhookconfigurations
# istio-sidecar-injector          (old, default)
# istio-sidecar-injector-1-31-1   (new)
``` 
At this point nothing has changed for your workloads. They still use the old istiod.

## Step 3: Switch the namespace to the new revision

The istio-injection label takes precedence over istio.io/rev, so you have to remove it. Otherwise pods keep getting the old proxy.

```bash
kubectl label namespace default istio-injection- istio.io/rev=1-31-1
```

## Step 4: Restart the workloads (this is the canary moment)

You can restart a single deployment first if you want a true canary:

```bash
kubectl rollout restart deployment/webapp -n default
istioctl proxy-status
```
`webapp` should now show `istiod-1-31-1-...` with version `1.31.1`, and the rest should still be on 1.30. Test your app, and check that mixed 1.30 and 1.31 traffic works. When you’re happy, restart the rest:

```bash
kubectl rollout restart deployment -n default
istioctl proxy-status    # all default/* pods → 1.31.1
```
# Rollback (if anything breaks during steps 3 and 4)

```bash
kubectl label namespace default istio.io/rev- istio-injection=enabled
kubectl rollout restart deployment -n default
```
This works because the old istiod is still running.

## Step 5: Upgrade the gateways

Enable the gateways in your file by deleting the components: block or setting `enabled: true`, then reapply it:

```bash
istioctl install -f canary-1-31-1.yaml -y
```
This makes the gateways owned by revision 1-31-1 and rolls them to 1.31. Gateways are normally upgraded in place like this. Running a canary of the gateway itself would mean a second gateway deployment and shifting traffic between them, which is overkill for a lab. Check the result:

```bash
istioctl proxy-status   # istio-ingressgateway → istiod-1-31-1, 1.31.1
```
## Step 6: Remove the old control plane

Run this without -y so you can read the list of resources it plans to delete before confirming. Make sure the gateways are not on that list. They shouldn’t be after step 5.

```bash
istioctl uninstall --revision default
```
## Step 7: Make 1.31 the default revision

This makes istio-injection=enabled work again for any new namespaces you create:

```bash
istioctl tag set default --revision 1-31-1
kubectl get pods -n istio-system
istioctl proxy-status
istioctl analyze -A
```

## Note on ambient mode
The control plane (istiod) can be canaried with revisions the same way, but **ztunnel** is a per-node DaemonSet and is upgraded differently. Check the official ambient upgrade guide before upgrading an ambient mesh.

---
