# redis-gitops

ArgoCD source repository for the Redis (app + db tier) deployment on
`lab.ocp.local`, managed via OpenShift GitOps.

Full design (LLD), review history, and rationale live in
[`OCP_Issue-Fix_RCA` issue 15](https://github.com/esarath/OCP_Issue-Fix_RCA/blob/main/issues/15-redis-app-db-gitops-deployment/Redis-OCP-GitOps-LLD.md) —
this repo holds only the manifests ArgoCD actually syncs.

## Layout

```
redis-gitops/
├── clusters/
│   └── ocp-onprem/
│       └── redis-project.yaml       # AppProject
└── apps/
    ├── redis-platform/              # namespace + governance (sync-wave 0)
    │   ├── application.yaml
    │   ├── base/                    # ArgoCD-managed: namespace, NetworkPolicy
    │   └── manual/                  # NOT ArgoCD-managed: ResourceQuota, LimitRange
    │                                 # (platform RBAC blocks the controller from
    │                                 # creating these — see manual/README.md)
    ├── redis-app/                   # cache tier (sync-wave 1)
    │   ├── application.yaml
    │   ├── base/                    # Deployment, Service, PodDisruptionBudget
    │   └── overlays/dev/            # what redis-app-appl actually points to
    └── redis-db/                    # db tier (sync-wave 1)
        ├── application.yaml
        ├── base/                    # StatefulSet, Service, PodDisruptionBudget
        └── overlays/dev/            # what redis-db-appl actually points to
```

All three Applications (`redis-platform-appl`, `redis-app-appl`, `redis-db-appl`)
and their manifests are present. `apps/redis-platform/manual/` still needs a
one-time `oc apply` — see that folder's README.

Validate any tree renders cleanly before applying:
```bash
oc kustomize apps/redis-app/overlays/dev
oc kustomize apps/redis-db/overlays/dev
```
