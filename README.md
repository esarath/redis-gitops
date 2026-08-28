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
    ├── redis-app/                   # cache tier (sync-wave 1) — TODO
    └── redis-db/                    # db tier (sync-wave 1) — TODO
```

`apps/redis-app/` and `apps/redis-db/` aren't populated yet — see the LLD's
Phase E/F for what belongs there.
