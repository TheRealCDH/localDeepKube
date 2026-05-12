# localDeepKube

**Deploy Local Deep Research on Kubernetes with Flux — zero config.**

Single Kustomize repo, single `agent` namespace. Everything pre-wired.

| Service | Role | In-cluster address |
|---|---|---|
| **Local Deep Research** | AI research assistant web UI | `http://local-deep-research:5000` |
| **LiteLLM** | Central LLM gateway → OpenRouter | `http://litellm:4000` |
| **SearXNG** | Private meta-search engine (API-only, no UI) | `http://searxng:8080` |

## Architecture

```
              ┌──────────────────────────────────────┐
              │            Tailnet (Tailscale)         │
              │   http://local-deep-research           │
              └──────────┬───────────────────────────┘
                         │
              ┌──────────▼───────────────────────────┐
              │   namespace: agent                    │
              │                                      │
              │  local-deep-research:5000             │
              │    LLM ──────► litellm:4000           │
              │    Search ───► searxng:8080           │
              │                                      │
              │  LiteLLM ──► OpenRouter (cloud)       │
              └──────────────────────────────────────┘
```

**Key design decisions:**
- **No Ollama** — all LLM calls go to OpenRouter via LiteLLM. No GPU needed.
- **No FlareSolverr** — LDR handles web fetching natively with Playwright.
- **Single API key** — OpenRouter key lives in one Secret, never seen by other services.
- **SearXNG is backend-only** — no search UI, no registration, just JSON API for LDR.
- **Single namespace** — everything in `agent`, services resolve by short name.

## Prerequisites

- Kubernetes 1.28+
- [Tailscale operator](https://tailscale.com/kb/1431/install-kubernetes-operator) installed in cluster
- Flux CD (or just `kubectl apply -k`)
- An [OpenRouter](https://openrouter.ai/) API key

## Quick Start

### 1. Create the API key secret

```bash
kubectl create namespace agent

kubectl create secret generic litellm-secrets \
  --namespace agent \
  --from-literal=openrouter-api-key=sk-or-v1-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

This is the **only** place the key lives in the cluster.

### 2. Deploy

```bash
kubectl apply -k .
```

Or with Flux:

```bash
flux create source git localdeepkube \
  --url=https://github.com/<your-org>/localDeepKube \
  --branch=main

flux create kustomization localdeepkube \
  --source=localdeepkube \
  --path=./ \
  --prune=true \
  --interval=5m
```

### 3. Expose via Tailscale

```bash
kubectl apply -f tailscale-service.yaml
```

Access `http://local-deep-research-tailscale` on your tailnet.

Or use the `TailscaleService` CRD:

```yaml
apiVersion: tailscale.com/v1alpha1
kind: TailscaleService
metadata:
  name: local-deep-research
  namespace: agent
spec:
  service: local-deep-research
```

### 4. First login

1. Visit the tailnet URL
2. Register a new account (registration is enabled)
3. Settings → LLM Provider already points to `http://litellm:4000/v1`
4. Start researching

**No API key to paste** — OpenRouter is handled entirely by LiteLLM.

## LDR settings (applied automatically)

| Variable | Value |
|---|---|
| `LDR_LLM_PROVIDER` | `openai_endpoint` |
| `LDR_LLM_OPENAI_ENDPOINT_URL` | `http://litellm:4000/v1` |
| `LDR_LLM_MODEL` | `deepseek-v4-flash` |
| `LDR_SEARCH_ENGINE_WEB_SEARXNG_DEFAULT_PARAMS_INSTANCE_URL` | `http://searxng:8080` |
| `LDR_APP_ALLOW_REGISTRATIONS` | `true` |

## Adding models

Edit `litellm/config.yaml` and add to `model_list`:

```yaml
  - model_name: my-model
    litellm_params:
      model: openrouter/provider/model-id
      api_key: os.environ/OPENROUTER_API_KEY
```

Push — Flux rolls the pod automatically.

## Using from other services

Any service in the `agent` namespace can reach LiteLLM at `http://litellm:4000/v1`. No auth required within the cluster.

```bash
curl http://litellm:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "deepseek-v4-flash", "messages": [{"role": "user", "content": "Hello"}]}'
```

## Repo structure

```
localDeepKube/
├── kustomization.yaml              # Root — includes all + namespace
├── namespace.yaml                  # Creates `agent` namespace
├── tailscale-service.yaml          # Tailscale exposure for LDR Web UI
├── litellm/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml                # ClusterIP :4000
│   └── config.yaml                 # Model allowlist → OpenRouter
├── searxng/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml                # ClusterIP :8080
│   └── settings.yml                # Private, API-only, engines enabled
└── local-deep-research/
    ├── kustomization.yaml
    ├── deployment.yaml
    ├── service.yaml                # ClusterIP :5000
    └── pvc.yaml                    # 10Gi for /data
```

## Default models in LiteLLM

| Model name | OpenRouter model | Use case |
|---|---|---|
| `deepseek-v4-flash` | `deepseek/deepseek-v4-flash` | Default — fast, cheap |
| `claude-sonnet-4` | `anthropic/claude-sonnet-4-20250514` | Deep reasoning |
| `claude-sonnet-4-thinking` | `anthropic/claude-sonnet-4-20250514` | Extended thinking |
| `gpt-4o` | `openai/gpt-4o` | General purpose |
| `qwen-32b` | `qwen/qwen-3-32b` | Local-style, cheap |
| `gemini-flash` | `google/gemini-2.5-flash-preview-04-17` | Very cheap, fast |
