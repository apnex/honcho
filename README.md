# honcho — self-hosted memory platform on k3s

GitOps manifests for [Honcho](https://github.com/plastic-labs/honcho) deployed via the `services` ApplicationSet in [apnex/labops](https://github.com/apnex/labops).

## What this deploys

| Component  | Purpose                                                    |
|------------|------------------------------------------------------------|
| postgres   | pgvector/pg17, 10Gi PVC (local-path), holds memory + embeds|
| redis      | Hot-path cache, 128MB max, ephemeral                       |
| honcho     | API server + background deriver (one Deployment)           |
| vip-honcho | MetalLB LoadBalancer on 192.168.1.250:8000                 |

## LLM backend

All Honcho text-gen routes through the existing LiteLLM proxy used by Hermes.
- Fast features (deriver, summary, low/medium dialectic): `smart-fast`
- Reasoning (high/max dialectic): `smart-reasoning`
- Dream consolidation: **disabled** initially to avoid surprise spend

## Image

Built from upstream `plastic-labs/honcho` and pushed to `ghcr.io/apnex/honcho:<version>`.
To rebuild for a new upstream release:

```sh
VER=v3.0.8
git clone --depth 1 -b $VER https://github.com/plastic-labs/honcho /tmp/honcho-build
cd /tmp/honcho-build
docker build -t ghcr.io/apnex/honcho:$VER -t ghcr.io/apnex/honcho:latest .
gh auth token | docker login ghcr.io -u apnex --password-stdin
docker push ghcr.io/apnex/honcho:$VER
docker push ghcr.io/apnex/honcho:latest
# Then bump manifests/honcho/deployment.yaml `image:` tag, commit, push.
```

## Secrets

The LiteLLM API key is **not** stored in this repo. It is applied out-of-band:

```sh
kubectl -n honcho create secret generic honcho-llm-keys \
  --from-literal=LLM_OPENAI_API_KEY=sk-...
```

The placeholder `Secret` in `manifests/honcho/secret.yaml` exists only so Argo
knows the resource is intentional. It will not be overwritten on sync.

If/when the cluster outgrows trust-the-network auth, set:
```
AUTH_USE_AUTH: "true"
AUTH_JWT_SECRET: <generate via scripts/generate_jwt_secret.py in upstream repo>
```

## Smoke test

```sh
curl -s http://192.168.1.250:8000/health        # → {"status":"ok"} or similar
kubectl -n honcho logs deploy/honcho --tail=50  # check no startup errors
```

## Wired into the cluster via

`apnex/labops` → `argo/services.yaml`:
```yaml
- name: honcho
  type: git
  repoURL: https://github.com/apnex/honcho
  gitPath: manifests
  revision: main
  namespace: honcho
```
