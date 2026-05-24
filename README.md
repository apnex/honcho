# honcho — self-hosted memory platform on Kubernetes

GitOps manifests for [Honcho](https://github.com/plastic-labs/honcho), deployed via
Kustomize. Designed to be portable to any Kubernetes cluster with a default
StorageClass and a LoadBalancer implementation (MetalLB, cloud LB, etc.).

## What this deploys

| Component         | Purpose                                                      |
|-------------------|--------------------------------------------------------------|
| postgres          | pgvector/pg17, 10Gi PVC, holds memory + embeddings           |
| redis             | Hot-path cache, 128MB max, ephemeral                         |
| honcho (API)      | FastAPI server, exposes `/v1/...`                            |
| honcho-deriver    | Background queue worker (`python -m src.deriver`)            |
| vip-honcho        | LoadBalancer Service on port `8000` (IP assigned by your LB) |

> **Note:** Honcho v3.x split the API server and the background deriver into
> separate processes. Both are required — the deriver builds representations
> and answers `dialectic` calls. Earlier docs referencing "one Deployment" are
> stale; the current truth is two Deployments sharing the same image + env.

## Image

The manifests reference `localhost/honcho:v3.0.7` with `imagePullPolicy: Never`.
This is the single-node convention: build once on the host, import into the
container runtime, no registry needed.

To rebuild for a new upstream release:

```sh
VER=v3.0.8
git clone --depth 1 -b $VER https://github.com/plastic-labs/honcho /tmp/honcho-build
cd /tmp/honcho-build
docker build -t localhost/honcho:$VER .

# Import into your container runtime:
# - k3s:        docker save ... | sudo k3s ctr -n k8s.io images import -
# - kind:       kind load docker-image localhost/honcho:$VER
# - microk8s:   docker save ... | microk8s ctr -n k8s.io images import -
docker save localhost/honcho:$VER | sudo k3s ctr -n k8s.io images import -

# Then bump the tag in manifests/honcho/deployment.yaml + deriver-deployment.yaml,
# commit, push.
```

**For multi-node clusters**, replace this flow with a registry push
(`ghcr.io/<you>/honcho:$VER` or similar) and drop `imagePullPolicy: Never`.

## Prerequisites

- Kubernetes cluster (k3s, k0s, microk8s, kind, EKS, GKE, AKS — any flavor)
- A default StorageClass (k3s ships `local-path`; cloud LBs ship their own)
- A LoadBalancer controller (MetalLB on-prem; cloud LBs are automatic)
- An OpenAI-compatible LLM endpoint with **tool-calling support**
  (LiteLLM router, OpenAI, OpenRouter, etc.) — verify with the curl probe
  in `docs/runbook.md`

## Install

### 1. Create the LLM key Secret (out-of-band)

```sh
kubectl create namespace honcho
kubectl -n honcho create secret generic honcho-llm-keys \
  --from-literal=LLM_OPENAI_API_KEY=sk-your-key-here
```

> The Secret is **not** in this repo — see `docs/secrets.md` for why
> (ArgoCD `selfHeal` would stomp a placeholder Secret on every reconcile).

### 2. Edit the LLM endpoint in the ConfigMap

In `manifests/honcho/configmap.yaml`, set:
```yaml
LLM_OPENAI_API_BASE: https://your-llm-router/v1
DERIVER_MODEL_CONFIG__OVERRIDES__BASE_URL: https://your-llm-router/v1
# ... (and any other model-config overrides for your endpoint)
```

### 3. Deploy

**Pick one.**

**(a) Direct — no GitOps:**
```sh
kubectl apply -k manifests/
```

**(b) With ArgoCD:**
```sh
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: honcho
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/apnex/honcho
    targetRevision: main
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: honcho
  syncPolicy:
    automated: { selfHeal: true, prune: true }
    syncOptions: [CreateNamespace=true]
EOF
```

### 4. Smoke test

Discover the LoadBalancer's assigned IP:
```sh
kubectl -n honcho get svc vip-honcho
# NAME         TYPE           EXTERNAL-IP      PORT(S)
# vip-honcho   LoadBalancer   <your-vip-ip>    8000:31xxx/TCP
```

Hit health:
```sh
VIP=$(kubectl -n honcho get svc vip-honcho -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s http://$VIP:8000/health        # → {"status":"ok"} or similar
kubectl -n honcho logs deploy/honcho --tail=50           # check no startup errors
kubectl -n honcho logs deploy/honcho-deriver --tail=50   # confirm worker is consuming
```

## Wired into the cluster via

Reference Argo wiring (for the `apnex/labops` ApplicationSet pattern):

```yaml
# argo/services.yaml
- name: honcho
  type: git
  repoURL: https://github.com/apnex/honcho
  gitPath: manifests
  revision: main
  namespace: honcho
```

## Operations

- Runbook (logs, restart, key rotation, backup/restore): `docs/runbook.md`
- Secrets management: `docs/secrets.md`
- Teardown: `docs/teardown.md`

## Long-term auth posture

For exposed-to-internet deployments, flip to JWT-gated mode:
```yaml
AUTH_USE_AUTH: "true"
AUTH_JWT_SECRET: <generate via upstream's scripts/generate_jwt_secret.py>
```
