# honcho teardown

Symmetric to install. Removes everything this repo deploys, with explicit
choices for data preservation vs. full wipe.

## What gets removed

| Resource                | Created by                       | Removed by         |
|-------------------------|----------------------------------|--------------------|
| Deployments (api + deriver) | `kubectl apply -k manifests/` | step 1             |
| Services (clusterIP + vip)  | `kubectl apply -k manifests/` | step 1             |
| ConfigMap (honcho-config)   | `kubectl apply -k manifests/` | step 1             |
| Postgres StatefulSet + PVC  | `kubectl apply -k manifests/` | step 1 (Set) + step 3 (PVC) |
| Redis Deployment            | `kubectl apply -k manifests/` | step 1             |
| `honcho-llm-keys` Secret    | manual `kubectl create secret`   | step 2             |
| `honcho` namespace          | `kubectl create namespace honcho` or Argo `CreateNamespace=true` | step 4 |
| ArgoCD Application (if used)| step 3 of Install                | step 0 (do this **first**) |

## Teardown — choose your path

### Path A: With ArgoCD

```sh
# 0. Delete the Application FIRST — Argo will reconcile-prune the workloads.
#    With `automated.prune: true`, this also removes everything the App manages.
kubectl -n argocd delete application honcho

# Wait for the prune to settle (Argo will delete Deployments, Services,
# ConfigMap, postgres StatefulSet, redis Deployment).
kubectl -n honcho get all     # should be empty (or just terminating pods)
```

Then continue from step 2 below to clean up the Secret + PVC + namespace.

### Path B: Direct (no GitOps)

```sh
# 1. Delete everything Kustomize created
kubectl delete -k manifests/    # or: kubectl -n honcho delete deploy,svc,cm,statefulset,secret -l app.kubernetes.io/part-of=honcho
```

## Common cleanup (both paths)

```sh
# 2. Delete the out-of-band LLM key Secret (not in Git, not managed by Argo/Kustomize)
kubectl -n honcho delete secret honcho-llm-keys --ignore-not-found

# 3. Delete the postgres PVC — DESTROYS ALL MEMORY DATA
#    Skip this step to preserve memory across a redeploy.
kubectl -n honcho delete pvc -l app=postgres
# Or by name: kubectl -n honcho delete pvc postgres-data-postgres-0

# 4. Delete the namespace (also removes any stray resources)
kubectl delete namespace honcho

# 5. If your StorageClass is `local-path`, the underlying data directory on
#    the host node remains by default (k3s convention). Purge it manually
#    if you want a truly clean slate:
sudo rm -rf /opt/local-path-provisioner/*pvc-*honcho*

# 6. If you used MetalLB and want the VIP back in the pool immediately,
#    no action needed — deleting the Service released the IP automatically.
```

## Verification

```sh
kubectl get ns honcho                       # → NotFound
kubectl get pv | grep honcho                # → empty (after step 3)
kubectl -n argocd get app honcho            # → NotFound (after step 0)
```

## Reinstall after teardown

Follow `README.md` install steps from scratch. If you skipped step 3 (PVC
deletion), the new deployment will attach to the existing PVC and recover
prior memory data on first start.

## Caveats

- **In-flight queue items are lost.** The deriver queue lives in Postgres;
  if you delete the PVC, anything not yet processed is gone.
- **Hermes will 5xx on memory calls** while honcho is down. The Honcho
  plugin in Hermes does not fall back to a no-memory mode — it errors.
  Set `enabled: false` in `honcho.json` if you want Hermes to keep running
  cleanly during the teardown window.
- **Argo App deletion is async.** If pods are stuck terminating, check for
  finalizers: `kubectl -n honcho get pods -o jsonpath='{.items[*].metadata.finalizers}'`.
