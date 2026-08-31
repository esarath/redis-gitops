# redis-gitops

ArgoCD source repository for the Redis platform namespace/governance on
`lab.ocp.local`, managed via OpenShift GitOps.

Full design (LLD), review history, and rationale live in
[`OCP_Issue-Fix_RCA` issue 15](https://github.com/esarath/OCP_Issue-Fix_RCA/blob/main/issues/15-redis-app-db-gitops-deployment/Redis-OCP-GitOps-LLD.md) —
this repo holds only the manifests ArgoCD actually syncs.

**2026-08-31: `redis-app` and `redis-db` (the app/db tiers) were fully torn
down** — workloads, PVCs, ArgoCD Applications, and manifests all removed. See
[`docs/redis-app-db-teardown.md`](docs/redis-app-db-teardown.md) for the
full runbook and rationale (freeing capacity to test the new
`ocp-gitops-poc` multi-tenancy onboarding flow with a fresh redis
app/db deployed into its `stage` tenant namespace instead). Only
`redis-platform` (namespace + governance + monitoring) remains.

## Layout

```
redis-gitops/
├── clusters/
│   └── ocp-onprem/
│       └── redis-project.yaml       # AppProject
├── docs/
│   └── redis-app-db-teardown.md     # 2026-08-31 teardown runbook
└── apps/
    └── redis-platform/              # namespace + governance (sync-wave 0)
        ├── application.yaml
        ├── base/                    # ArgoCD-managed: namespace, NetworkPolicy, monitoring
        └── manual/                  # NOT ArgoCD-managed: ResourceQuota, LimitRange
                                       # (platform RBAC blocks the controller from
                                       # creating these — see manual/README.md)
```

Only `redis-platform-appl` remains. `apps/redis-platform/manual/` still needs
a one-time `oc apply` — see that folder's README.

Validate the tree renders cleanly before applying:
```bash
oc kustomize apps/redis-platform/base
```
