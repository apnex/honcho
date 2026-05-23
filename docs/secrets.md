# Secrets management

The `honcho-llm-keys` Secret (containing `LLM_OPENAI_API_KEY`) is **not** in
this repository — at all. It is created manually:

```sh
kubectl -n honcho create secret generic honcho-llm-keys \
  --from-literal=LLM_OPENAI_API_KEY=sk-...
```

## Why not a placeholder Secret in Git?

We tried that initially with `argocd.argoproj.io/compare-options: IgnoreExtraneous`,
but Argo's `IgnoreExtraneous` only suppresses warnings about resources Argo
doesn't know about — it does not stop Argo from overwriting a resource it
*does* manage. The placeholder would silently stomp the real key on every
reconciliation.

## To rotate the key

```sh
kubectl -n honcho delete secret honcho-llm-keys
kubectl -n honcho create secret generic honcho-llm-keys \
  --from-literal=LLM_OPENAI_API_KEY=sk-NEW
kubectl -n honcho rollout restart deploy/honcho
```

## Long-term

When ready, replace this manual flow with:
- **Sealed Secrets** (commit encrypted secrets to git, controller decrypts in-cluster)
- **External Secrets Operator** (sync from Vault/AWS Secrets Manager/etc.)
- **SOPS + Kustomize** (encrypt-at-rest values in Git)
