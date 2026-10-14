# Istio Canary Upgrade with istioctl

A guide to upgrading Istio with the **canary (revision-based)** method: run the new control plane next to the old one, move workloads over gradually, then remove the old control plane.

Compared to an in-place upgrade (`istioctl upgrade`), a canary upgrade lets you test the new version on a small part of the mesh first and roll back by changing a label.


## 1. How it works

### Injection

When a pod is **created**, Kubernetes asks mutating webhooks whether they want to modify it. Istio's webhook adds the Envoy sidecar. The sidecar version is the version of the istiod whose webhook handled the pod.

> Injection only happens at pod creation. Changing labels or tags does nothing to running pods: they must be restarted.

### Revisions

A revision is the **name of an istiod installation**. Several revisions can run side by side.

| Install command | Revision | istiod Deployment |
|---|---|---|
| `istioctl install` | `default` | `istiod` |
| `istioctl install --set revision=1-31-1` | `1-31-1` | `istiod-1-31-1` |

Revision names can't contain dots, so `1.31.1` becomes `1-31-1`.

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



### The upgrade at a glance

```
1. Install new istiod as a revision      old istiod ✔   new istiod ✔   workloads → old
2. Relabel one namespace + restart       old istiod ✔   new istiod ✔   some → new (canary)
3. Relabel the rest + restart            old istiod ✔   new istiod ✔   all → new
4. Upgrade gateways                      old istiod ✔   new istiod ✔   gateways → new
5. Uninstall old istiod                                 new istiod ✔   all → new
```

Old and new proxies work together during the migration: both istiods share the same root CA, so mTLS keeps working across versions.

---

## 2. Variables used in this guide

```bash
export OLD_REV=default     # use "default" if the current install has no revision,
                           # otherwise its revision name, e.g. 1-30-5
export NEW_REV=1-31-1      # revision name for the new version
export NS=my-namespace     # a namespace to migrate
```

Find the current revision:

```bash
kubectl get pods -n istio-system -l app=istiod --show-labels   # look at istio.io/rev
```

> Use the **new** `istioctl` for every step. `istioctl` installs the Istio version that matches its own version.

```bash
istioctl version --remote=false   # must print the new version
```

To download a specific version:

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.31.1 sh -
export PATH="$PWD/istio-1.31.1/bin:$PATH"
```

---

## 3. Pre-flight checks

```bash
istioctl x precheck      # is the cluster ready for the new version?
istioctl analyze -A      # fix existing problems before upgrading
```

Read the upgrade notes for the target version on https://istio.io/latest/news/releases/ and make sure the jump is supported (usually one minor version at a time, e.g. 1.30 → 1.31).

### Capture the current configuration

The new revision gets its **own** mesh config. Any customization that isn't carried over is silently lost for migrated workloads.

```bash
# Current mesh config of the old revision
kubectl get configmap istio -n istio-system -o jsonpath='{.data.mesh}'                 # OLD_REV=default
kubectl get configmap istio-$OLD_REV -n istio-system -o jsonpath='{.data.mesh}'        # revisioned
```

Also note the profile and any `--set` flags or IstioOperator file used for the original install.

---

## 4. Install the new control plane

### Create an overlay file

```yaml
# upgrade.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default          # same profile as the current install
  revision: 1-31-1          # NEW_REV
  meshConfig: {}            # copy custom mesh settings here
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: false        # gateways are moved later (section 6)
```

> The file is an **overlay**: `istioctl` loads the full profile and merges the file on top. You get a complete control plane with your changes, not only what's in the file.

Disabling the gateway matters when the gateway was installed by `istioctl`. Otherwise the new install reuses the same Deployment name and upgrades the gateway **in place immediately**, before the new control plane has been tested. If the profile includes an egress gateway, disable it the same way under `egressGateways`.

### Preview

```bash
istioctl manifest generate -f upgrade.yaml | grep -E '^kind:|^  name:' | paste - - | sort | uniq
```

You should see `istiod-<NEW_REV>` resources and no gateway.

### Install

```bash
istioctl install -f upgrade.yaml -y
```

This creates:

- `istiod-<NEW_REV>` (Deployment, Service, ConfigMaps, RBAC)
- webhook `istio-sidecar-injector-<NEW_REV>`
- updated CRDs and cluster resources (shared and backward compatible)

The old istiod, its webhook and the gateways are not touched.

### Verify

```bash
kubectl get pods -n istio-system -l app=istiod --show-labels
kubectl get mutatingwebhookconfigurations
```

Both revisions should be running. No workload has changed yet.

---

## 5. Migrate workloads

### List namespaces and their labels

```bash
kubectl get ns -L istio-injection -L istio.io/rev
```

### Relabel a namespace

```bash
# If it uses istio-injection=enabled
kubectl label namespace $NS istio-injection- istio.io/rev=$NEW_REV

# If it already uses istio.io/rev
kubectl label namespace $NS istio.io/rev=$NEW_REV --overwrite
```

### Canary: restart one deployment first

```bash
kubectl rollout restart deployment/<name> -n $NS
istioctl proxy-status
```

The restarted pod should show `istiod-<NEW_REV>` and the new version. Test it: traffic, mTLS, telemetry, policies.

### Then the rest of the namespace

```bash
kubectl rollout restart deployment -n $NS
# also statefulsets / daemonsets if the namespace has them
kubectl rollout restart statefulset -n $NS
kubectl rollout restart daemonset -n $NS
```

Repeat for every injected namespace, at your own pace.

### Check progress

```bash
istioctl proxy-status          # ISTIOD and VERSION columns per proxy
```

---

## 6. Upgrade the gateways

### Gateways installed by istioctl

Re-enable the gateway in `upgrade.yaml` (remove the `components:` block or set `enabled: true`) and apply again:

```bash
istioctl install -f upgrade.yaml -y
istioctl proxy-status          # gateway → istiod-<NEW_REV>
```

The gateway is upgraded **in place** (rolling update) and is now owned by the new revision. This is the usual approach for gateways.

### Gateways installed separately (Helm or gateway injection)

These use injection like normal pods. Point them at the new revision and restart:

```bash
kubectl label namespace <gateway-ns> istio.io/rev=$NEW_REV --overwrite
kubectl rollout restart deployment/<gateway> -n <gateway-ns>
```

For a true canary of a gateway, deploy a second gateway on the new revision with the same selector labels, shift traffic, then remove the old one.

---

## 7. Remove the old control plane

Only when:

- `istioctl proxy-status` shows **every** proxy on `istiod-<NEW_REV>`
- no namespace still depends on the old revision (including `istio-injection=enabled` if you won't set a default tag, see section 8)
- gateways are on the new revision

Run **without `-y`** and read the list of resources before confirming:

```bash
istioctl uninstall --revision $OLD_REV
```

Then verify:

```bash
kubectl get pods -n istio-system
istioctl version
istioctl analyze -A
```

---

## 8. Default tag or explicit labels?

After the migration, decide how namespaces should reference the control plane.

### Option A: explicit revision labels, no default tag

Every namespace uses `istio.io/rev=<revision>`.

- Full per-namespace control on every upgrade.
- New namespaces need the explicit label; `istio-injection=enabled` does nothing once the old default revision is gone.
- Apply-time validation of Istio resources without a revision label may be weaker. Run `istioctl analyze -A` after changes.

### Option B: set the default tag

```bash
istioctl tag set default --revision $NEW_REV
istioctl tag list
```

This:

- makes `istio-injection=enabled` (and `istio.io/rev=default`) inject the new revision
- moves Istio config validation to the new revision
- deactivates the old default injection webhook if it still exists
- restarts nothing: only new pods are affected

`istio.io/rev=<NEW_REV>` labels keep working.

> Don't set the default tag in the middle of a migration. Every namespace still on `istio-injection=enabled` would silently switch on its next restart.

### Option C: stable custom tags (recommended for long-term use)

Label namespaces with a tag instead of a version:

```bash
istioctl tag set prod-stable --revision $NEW_REV
kubectl label namespace $NS istio-injection- istio.io/rev=prod-stable --overwrite
```

Next upgrade, move the tag instead of relabeling namespaces:

```bash
istioctl tag set prod-canary --revision <next-rev>      # test on a few namespaces first
istioctl tag set prod-stable --revision <next-rev> --overwrite
kubectl rollout restart deployment -n $NS
```

### Comparison

| Namespace label | Next upgrade |
|---|---|
| `istio.io/rev=<revision>` | relabel each namespace + restart |
| `istio-injection=enabled` | move the `default` tag + restart (all namespaces at once) |
| `istio.io/rev=<custom-tag>` | move the tag + restart (per group of namespaces) |

---

## 9. Rollback

Possible as long as the old control plane is still installed (before section 7).

```bash
# Back to a non-revisioned default install
kubectl label namespace $NS istio.io/rev- istio-injection=enabled

# Back to a revisioned install
kubectl label namespace $NS istio.io/rev=$OLD_REV --overwrite

kubectl rollout restart deployment -n $NS
istioctl proxy-status
```

To roll back a gateway that was upgraded in place, re-run the **old** `istioctl install` with the original configuration.

To remove the new control plane entirely:

```bash
istioctl uninstall --revision $NEW_REV
```

---

## 10. Command reference

```bash
istioctl version                                   # client / control plane / data plane
istioctl version --remote=false                    # client only
istioctl x precheck                                # upgrade readiness
istioctl analyze -A                                # config problems in all namespaces
istioctl proxy-status                              # istiod + version per proxy
istioctl tag list                                  # tags → revisions
istioctl manifest generate -f upgrade.yaml         # render YAML without applying
kubectl get ns -L istio-injection -L istio.io/rev  # namespace labels
kubectl get mutatingwebhookconfigurations          # injection webhooks
kubectl get pods -n istio-system -l app=istiod --show-labels
```

### The preview pipeline explained

```bash
istioctl manifest generate -f upgrade.yaml | grep -E '^kind:|^  name:' | paste - - | sort | uniq
```

| Part | What it does |
|---|---|
| `istioctl manifest generate -f upgrade.yaml` | renders the full install as YAML; applies nothing |
| `grep -E '^kind:\|^  name:'` | keeps `kind:` lines and `metadata.name` lines (two-space indent) |
| `paste - -` | joins every two lines: `kind: X    name: Y` |
| `sort \| uniq` | groups by kind and removes duplicates |

A quick shortcut, not a parser. For exact output use `yq`:

```bash
istioctl manifest generate -f upgrade.yaml | yq -N '.kind + "  " + .metadata.name' | sort
```

---

## 11. Gotchas

- **The `istioctl` version decides the installed version.** Check `which istioctl` and `istioctl version --remote=false` before installing. Package managers (e.g. Homebrew) may shadow a downloaded binary.
- **`istio-injection` beats `istio.io/rev`.** Remove it when moving a namespace to a revision.
- **Labels and tags only affect new pods.** Always restart.
- **Carry over custom `meshConfig`.** Access logs, tracing and other settings don't transfer automatically.
- **Disable istioctl-managed gateways in the first install** if you don't want them upgraded immediately.
- **Gateways belong to the revision that installed them.** Move them before uninstalling the old revision, or they may be deleted with it.
- **Run `istioctl uninstall` without `-y` first** and read what it will remove.
- **Don't set the default tag mid-migration.**
- **Check `istioctl proxy-status` before removing anything.**
