# redis-app / redis-db Teardown Runbook

**Date:** 2026-08-31
**Why:** clearing the way to test the `ocp-gitops-poc` multi-tenancy
self-service onboarding flow end-to-end by deploying a fresh redis app/db
into the new `stage` tenant namespace via ArgoCD, rather than running two
parallel redis demos. `redis-platform` (namespace, governance, monitoring)
was kept — only the `redis-app`/`redis-db` workload tiers were removed.

**Result:** complete — both Applications, all their live resources, the
untracked PVCs and auth Secrets, the MetalLB-allocated external IP, and the
source manifests in this repo are all gone. `redis-platform` namespace now
contains only `kube-state-metrics-redis` (platform monitoring).

## What was removed

| Resource | Kind | How it was tracked |
|---|---|---|
| `redis-app-appl` | ArgoCD Application | Git (`apps/redis-app/application.yaml`) |
| `redis-db-appl` | ArgoCD Application | Git (`apps/redis-db/application.yaml`) |
| `redis-app` | Deployment | ArgoCD-managed (cascade-deleted) |
| `redis-app` | Service (ClusterIP) | ArgoCD-managed |
| `redis-app-external` | Service (LoadBalancer, MetalLB IP `192.168.29.70`) | ArgoCD-managed |
| `redis-app` | PodDisruptionBudget | ArgoCD-managed |
| `allow-external-redis-test` | NetworkPolicy | ArgoCD-managed |
| `redis-db` | StatefulSet | ArgoCD-managed (replicas ignored via `ignoreDifferences`, to stop fighting the HPA) |
| `redis-db` | Service (headless) | ArgoCD-managed |
| `redis-db` | HorizontalPodAutoscaler | ArgoCD-managed |
| `redis-db` | PodDisruptionBudget | ArgoCD-managed |
| `data-redis-db-0`, `data-redis-db-1` | PersistentVolumeClaim (10Gi each, `nfs-storage`) | **Not** ArgoCD-tracked — StatefulSet `volumeClaimTemplates` PVCs always outlive the StatefulSet by default; deleted manually. **Irreversible — all Redis data on these volumes is gone.** |
| `redis-app-auth`, `redis-db-auth` | Secret | **Not** ArgoCD-tracked (created out-of-band, standard practice for not committing secrets to Git); deleted manually |
| `apps/redis-app/`, `apps/redis-db/` | Git source directories | This repo — removed entirely |

**Not removed** (out of this teardown's scope): `redis-platform` namespace,
its `NetworkPolicy`/`RoleBinding`/monitoring stack (`kube-state-metrics-redis`,
`ServiceMonitor`, `PrometheusRule`s), and the manually-applied
`ResourceQuota`/`LimitRange`.

## Known follow-up (not fixed here, out of scope for this teardown)

`apps/redis-platform/base/monitoring/redis-hpa-alerts.yaml` and
`prometheusrule.yaml` contain alert rules keyed to
`horizontalpodautoscaler="redis-db"` and `poddisruptionbudget="redis-app"`.
With those objects gone, the underlying metrics series simply stop
existing — the rules go quietly inert (no error, no false alarms), but
they're now dead references. Worth pruning next time that file is touched;
left alone here to keep this teardown scoped to redis-app/redis-db only.

## Steps actually run (in order)

1. **Confirm scope and current tracked resources** before touching anything:
   ```bash
   kubectl get application redis-app-appl redis-db-appl -n openshift-gitops -o json
   ```

2. **Add the cascade-delete finalizer** to both Applications (without this,
   deleting the `Application` object leaves its resources orphaned in the
   cluster instead of removing them):
   ```bash
   kubectl patch application redis-app-appl -n openshift-gitops --type merge \
     -p '{"metadata":{"finalizers":["resources-finalizer.argocd.argoproj.io"]}}'
   kubectl patch application redis-db-appl -n openshift-gitops --type merge \
     -p '{"metadata":{"finalizers":["resources-finalizer.argocd.argoproj.io"]}}'
   ```

3. **Delete the Application objects** — ArgoCD prunes every resource it
   tracks before the Application object itself is removed:
   ```bash
   kubectl delete application redis-app-appl -n openshift-gitops --wait=true --timeout=90s
   kubectl delete application redis-db-appl -n openshift-gitops --wait=true --timeout=90s
   ```

4. **Verify** the Deployment, both Services (including the LoadBalancer),
   StatefulSet, HPA, PDB, and NetworkPolicy are actually gone:
   ```bash
   kubectl get all -n redis-platform
   kubectl get networkpolicy -n redis-platform
   ```

5. **Delete the untracked PVCs** — StatefulSet `volumeClaimTemplates` PVCs
   are never owned/cleaned up automatically:
   ```bash
   kubectl delete pvc data-redis-db-0 data-redis-db-1 -n redis-platform
   ```

6. **Delete the untracked auth Secrets**:
   ```bash
   kubectl delete secret redis-app-auth redis-db-auth -n redis-platform
   ```

7. **Confirm the MetalLB IP was released**:
   ```bash
   kubectl get ipaddresspool -n metallb-system -o yaml   # assignedIPv4 back to 0
   ```

8. **Remove the source manifests from Git** and push:
   ```bash
   git rm -r apps/redis-app apps/redis-db
   git commit -m "..."
   git push
   ```

## Verification (all confirmed 2026-08-31)

- `kubectl get applications -n openshift-gitops` — only `redis-platform-appl`
  remains from this project; `app-of-apps`, `sample-app-*`, `tenant-*` from
  the separate `ocp-gitops-poc` project untouched.
- `kubectl get all,pvc,secret,networkpolicy -n redis-platform` — only
  `kube-state-metrics-redis` (Deployment/Pod/ReplicaSet/Service) and its
  `dockercfg` Secret remain, plus the platform NetworkPolicies
  (`default-deny-ingress`, `allow-same-namespace`, `allow-monitoring-scrape`).
- MetalLB `IPAddressPool` status: `assignedIPv4: 0` (was `1`), pool fully
  free again.
- No `Warning` events in `redis-platform` from the teardown itself.
