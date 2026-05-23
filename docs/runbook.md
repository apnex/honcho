# honcho runbook

## Common ops

### View logs
```
kubectl -n honcho logs deploy/honcho --tail=200 -f
kubectl -n honcho logs statefulset/postgres --tail=50
```

### Restart Honcho (config / secret change)
```
kubectl -n honcho rollout restart deploy/honcho
kubectl -n honcho rollout status  deploy/honcho
```

### Rotate the LiteLLM key
```
kubectl -n honcho delete secret honcho-llm-keys
kubectl -n honcho create secret generic honcho-llm-keys \
  --from-literal=LLM_OPENAI_API_KEY=sk-NEW
kubectl -n honcho rollout restart deploy/honcho
```

### Force Argo sync (after a repo push)
```
kubectl -n argocd patch application honcho --type=merge \
  -p '{"operation":{"sync":{"revision":"HEAD","prune":true}}}'
```

### Postgres backup (manual)
```
kubectl -n honcho exec statefulset/postgres -- \
  pg_dump -U honcho honcho | gzip > honcho-$(date +%F).sql.gz
```

### Restore
```
gunzip -c honcho-YYYY-MM-DD.sql.gz | \
  kubectl -n honcho exec -i statefulset/postgres -- psql -U honcho honcho
```

## Things to watch

- **Disk usage on the PVC.** Honcho stores embeddings — they grow with usage. Alert if PVC > 80%.
- **Deriver lag.** If the deriver queue stalls, memory stops updating. Check `kubectl -n honcho logs deploy/honcho | grep -i deriver`.
- **LiteLLM proxy budget.** Every Hermes turn produces ~3-10 background LLM calls.

## Re-enabling dream consolidation

Once stable, flip in `manifests/honcho/configmap.yaml`:
```
DREAM_ENABLED: "true"
```
…then commit, push, let Argo sync. Dreams fire at most every 8h by default.
