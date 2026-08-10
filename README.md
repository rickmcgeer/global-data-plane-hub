# global-data-plane-hub

A snapshot of the **live, currently-deployed** JupyterHub for the Global
Data Plane project, captured 2026-08-10 before any migration work touches
it - so none of this is lost regardless of what happens next.

- **Cluster**: `jupyter-ai-cluster`, project `civil-pattern-439416-k1`, zone
  `us-west2-a`
- **Release**: `jupyter-ai-1` (namespace `jupyter-ai`), chart
  `jupyterhub-4.0.0`, revision 10, deployed 2025-10-23
- **Hostname**: `bearborg.global-data-plane.org`

This is **not** the same effort as
[`jupyter-ai-image`](https://github.com/rickmcgeer/jupyter-ai-image) /
[`jupyter-ai-deployment`](https://github.com/rickmcgeer/jupyter-ai-deployment)
(the from-scratch Jupyter AI v3 rebuild) - those didn't know this cluster
existed when they were built. This repo exists to reconcile the two: decide
what here needs to survive a migration to the new architecture, and what's
safe to drop.

## What's actually running

| Resource | Image | Notes |
|---|---|---|
| `hub`, `proxy`, `autohttps`, `user-scheduler` | z2jh 4.0.0 stock | The Hub itself |
| `gdp-service` | `rickmcgeer/gdp-sidecar:20251023` | **The Global Data Plane - keep this.** Exposed at `/services/gdp` via Hub OAuth. |
| `litellm-deployment` + `postgres-deployment` | `ghcr.io/berriai/litellm:main-stable` | A **second, separate, self-hosted** LiteLLM proxy in-cluster (distinct from LBNL's external `bearborg.berkeley.edu:4433`). Exposed at `/services/litellm/`. |
| `hello-world-deployment` | `gcr.io/google-samples/hello-app:1.0` | Looks like a leftover smoke test, not load-bearing - not confirmed, don't assume. |
| singleuser pods | `rickmcgeer/jupyter-ai-2` / `jupyter-ai-3` | Three profiles: "BearBorg original", vanilla Datascience 3.11, "BearBorg (Custom) with AI support". |

## Three custom pieces glued into the Hub (`values.yaml`'s `extraConfig`)

1. **`bearborgHandler`** - serves [`configmaps/bearborg-static/index.html`](configmaps/bearborg-static/index.html)
   (a custom "Welcome to BearBorg" page) at `/hub/bearborg/`.
2. **`gdp-service-config`** - registers `gdp` as an OAuth-protected Hub
   service. **This is the piece to preserve.**
3. **`litellm_route`** - routes `/services/litellm/` to the in-cluster
   LiteLLM proxy.
4. **`test_env_config`** (`c.Spawner.pre_spawn_hook`) - on every spawn,
   calls LBNL's external LiteLLM proxy to auto-provision (or reuse) a
   per-user API key, then injects it as `LITELLM_API_KEY` /
   `OPENAI_API_KEY` / `OPENAI_API_BASE` into the singleuser pod. This is
   what the old `bearborg_ai` package (in the now-abandoned
   `bearborg-jupyter-ai` repo) consumed via those env vars - see
   [[project_berkeley_jupyter_ai_rearchitecture]] for why that package was
   dropped. Note this gives each user their own tracked/budgeted key
   automatically, with no manual provisioning - the new architecture's
   single shared Workload-Identity-backed key doesn't replicate that
   per-user accounting, if that mattered here.

Custom Hub UI templates (`home.html`, `login.html`, `spawn.html`) are in
[`configmaps/bearborg-custom-templates/`](configmaps/bearborg-custom-templates/)
- all three are ~7 lines, likely thin overrides rather than full rewrites.

## Secrets - deliberately NOT in this repo

`values.yaml` has every live secret replaced with a `<REDACTED: ...>`
placeholder and a pointer to where to actually get it (a `kubectl get
secret` command, or "ask LBNL science-it"). Do not fill these back in and
commit them. To see the real values against the live cluster:

```bash
gcloud container clusters get-credentials jupyter-ai-cluster --zone us-west2-a --project civil-pattern-439416-k1
helm get values jupyter-ai-1 -n jupyter-ai
kubectl get secret hub -n jupyter-ai -o yaml   # proxy secretToken, gdp api_token, etc.
```

## Redeploying from this snapshot

This isn't meant to be `helm upgrade`d from directly - it's a record. If
you do need to reproduce this exact deployment (e.g. as a rollback target):

```bash
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm upgrade --install jupyter-ai-1 jupyterhub/jupyterhub \
  --version 4.0.0 --namespace jupyter-ai \
  -f values.yaml \
  --set hub.config.GoogleOAuthenticator.client_secret=... \
  --set hub.extraConfig.gdp-service-config=...   # api_token substituted in
  # etc. - fill in every <REDACTED> placeholder above
```
