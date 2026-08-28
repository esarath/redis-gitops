# Manually-applied governance objects

`resourcequota.yaml` and `limitrange.yaml` live here, not in `../base/`,
because OpenShift GitOps' auto-granted namespace role for the
argocd-application-controller service account explicitly restricts
`resourcequotas`/`limitranges` to `get/list/watch` — it cannot `create` them,
even with `admin`-equivalent rights on everything else in the namespace.
This is a deliberate platform guardrail (a GitOps app can't grant itself its
own quota), not a bug.

Apply once, out-of-band, the same way secrets are handled in Phase D of the
LLD:

```bash
oc apply -f apps/redis-platform/manual/resourcequota.yaml
oc apply -f apps/redis-platform/manual/limitrange.yaml
```
