# Kueue hops

Hopsworks fork of Kueue. Per-release branches (`branch-0.11.3`, `branch-0.12.2`,
`branch-0.18.x`, ...) each carry the upstream release plus the customizations below,
packaged and pushed to `https://repo.hops.works/master/kueue/`.

## Customizations on top of upstream

### 1. CRDs consolidated into `crds/`

CRDs are rendered into a single static `charts/kueue/crds/crds.yaml` and the templated
`charts/kueue/templates/crd/` directory is removed:

```
cd charts/kueue
helm template . --include-crds --show-only 'templates/crd/*.yaml' > crds/crds.yaml
git rm -r templates/crd
```

Helm's `crds/` directory is applied directly at install and is **not** part of the
release manifest, so the (~1.6 MB) CRD set never lands in the Helm release Secret. The
templated layout would push it into the Secret and risk the 1 MiB cap when this chart is
a dependency of the Hopsworks umbrella chart.

### 2. Webhook object selector

Add to the `batch/job` mutate and validate webhooks in
`charts/kueue/templates/webhook/manifests.yaml` so only opted-in workloads are
intercepted (unlabelled workloads pass through, and a Kueue outage never blocks
unrelated Job creation):

```
objectSelector:
  matchLabels:
    app.kubernetes.io/managed-by: kueue
```

### 3. `enableVisibilityServerAuth` value

Gate the kube-system visibility RoleBinding (`templates/visibility/role_binding.yaml`)
behind `enableVisibilityServerAuth` (default `true`) so restricted / ArgoCD
namespace-scoped installs that cannot write to `kube-system` can opt out.

## Upgrade caveat (0.18+)

From 0.18 every CRD serves both `v1beta1` and `v1beta2` (storage = `v1beta2`), so the
CRD **conversion webhook is live** (on 0.12 each CRD served a single version and the
conversion webhook was never invoked). The consolidated `crds.yaml` freezes the
conversion `clientConfig` as `release-name-kueue-webhook-service` / namespace `default`,
and Kueue's internal cert controller injects only the `caBundle`, not the service
coordinates. This is harmless on a **fresh install** (everything is created and read at
`v1beta2`, so conversion is never triggered) but must be corrected for **in-place
upgrades**, where existing `v1beta1`-stored objects are converted:

- render/patch the conversion service name to `kueue-webhook-service`, and
- patch `.spec.conversion.webhook.clientConfig.service.namespace` to the release
  namespace at apply time (the CRD-apply / migration hook runs `kubectl` in-namespace).

Tracked with the upgrade migration hook work (HWORKS-2899).
