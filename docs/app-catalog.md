# Application catalog

This catalog is based on the Kubernetes and Argo CD manifests in this repository.

## Argo CD applications

| Application | Source | Destination | Notes |
| --- | --- | --- | --- |
| `apps` | `apps/` in this repo | in-cluster | App-of-apps root. Reconciles the other Argo CD `Application` objects. |
| `argocd` | `argocd/` in this repo | `argocd` | Argo CD installation and config. |
| `kube-system` | `kube-system/` in this repo | in-cluster | Cluster networking helpers and Hubble UI ingress. |
| `ingress` | `ingress/` in this repo | in-cluster | Shared ingress routes and certificates. |
| `homepage` | `homepage/` in this repo | `homepage` | Homepage dashboard. Uses kustomize config map generation. |
| `karakeep` | `karakeep/` in this repo | in-cluster | Karakeep web app plus Meilisearch and Chrome. |
| `tandoor` | `tandoor/` in this repo | `tandoor` | Tandoor Recipes plus PostgreSQL. |
| `glance` | `glance/` in this repo | in-cluster | Glance dashboard. |
| `foundryvtt` | `foundryvtt/` in this repo | in-cluster | Foundry VTT server. |
| `epic-free-games` | `epic-free-games/` in this repo | in-cluster | Epic Games Freegames Node workload. |
| `epic-free-games-ronin` | `epic-free-games-ronin/` in this repo | in-cluster | Second Epic Games Freegames Node workload. |
| `free-games-claimer` | `free-games-claimer/` in this repo | in-cluster | Free Games Claimer workload. |
| `free-games-claimer-ronin` | `free-games-claimer-ronin/` in this repo | in-cluster | Second Free Games Claimer workload. |
| `open-webui` | Helm chart `open-webui` from `https://helm.openwebui.com/` | `ai` | Open WebUI with pipelines enabled, embedded Ollama disabled, persistence enabled. |

All application manifests use automated sync with prune and self-heal enabled.

## Workloads and images

| Area | Workload | Namespace | Image(s) |
| --- | --- | --- | --- |
| `argocd` | Argo CD controllers, API server, repo server, Redis, Dex | `argocd` | `quay.io/argoproj/argocd:v2.13.2`, `redis:7.0.15-alpine`, `ghcr.io/dexidp/dex:v2.41.1` |
| `epic-free-games` | `Deployment/epic-free-games` | `epic-games-node` | `ghcr.io/claabs/epicgames-freegames-node:latest` |
| `epic-free-games-ronin` | `Deployment/epic-free-games` | `epic-games-node-ronin` | `ghcr.io/claabs/epicgames-freegames-node:latest` |
| `foundryvtt` | `Deployment/foundry-foundry-vtt` | `foundry` | `felddy/foundryvtt:13.350` |
| `free-games-claimer` | `Deployment/free-games-claimer` | `free-games-claimer` | `ghcr.io/vogler/free-games-claimer` |
| `free-games-claimer-ronin` | `Deployment/free-games-claimer` | `free-games-claimer-ronin` | `ghcr.io/vogler/free-games-claimer` |
| `glance` | `Deployment/glance` | `glance` | `glanceapp/glance:v0.8.4` |
| `homepage` | `Deployment/homepage` | `homepage` | `ghcr.io/gethomepage/homepage:latest` |
| `karakeep` | `Deployment/web` | `karakeep` | `ghcr.io/karakeep-app/karakeep:release` |
| `karakeep` | `Deployment/meilisearch` | `karakeep` | `getmeili/meilisearch:v1.11.1` |
| `karakeep` | `Deployment/chrome` | `karakeep` | `gcr.io/zenika-hub/alpine-chrome:124` |
| `tandoor` | `Deployment/recipes` | `tandoor` through Argo CD destination | `nginx:latest`, `vabene1111/recipes` |
| `tandoor` | `StatefulSet/recipes-postgresql` | `tandoor` through Argo CD destination | `docker.io/bitnami/postgresql:latest` |

## Ingress and external hosts

| Manifest | Kind | Namespace | Host |
| --- | --- | --- | --- |
| `ingress/coroot-traefik.yaml` | `IngressRoute` + `Certificate` | `coroot` | `coroot.mizar.scalar.cloud` |
| `ingress/foundryvtt.yaml` | `IngressRoute` + `Certificate` | `foundry` | `foundry.mizar.scalar.cloud` |
| `ingress/karakeep-traefik.yaml` | `IngressRoute` + `Certificate` | `karakeep` | `keep.mizar.scalar.cloud` |
| `ingress/open-webui-traefik.yaml` | `IngressRoute` + `Certificate` | `ai` | `ai.mizar.scalar.cloud` |
| `ingress/tandoor-traefik.yaml` | `IngressRoute` + `Certificate` | `tandoor` | `meals.mizar.scalar.cloud` |
| `ingress/argocd.yaml` | `Ingress` | `argocd` | Host is not set in the manifest. |
| `ingress/epic-free-games.yaml` | `Ingress` | `epic-games-node` | Host is not set in the manifest. |
| `ingress/epic-free-games-ronin.yaml` | `Ingress` | `epic-games-node-ronin` | Host is not set in the manifest. |
| `ingress/glance.yaml` | `Ingress` | `glance` | Host is not set in the manifest. |
| `ingress/homepage.yaml` | `Ingress` | `homepage` | Host is not set in the manifest. |
| `ingress/octobot.yaml` | `Ingress` | `crypto` | Host is not set in the manifest. |
| `kube-system/ingress.yaml` | `Ingress` | `kube-system` | Host is not set in the manifest. |

The Traefik routes use `websecure` and cert-manager `Certificate` resources. Keep the route host, certificate DNS name, and TLS secret name aligned when adding or renaming an endpoint.

## Storage

| Claim | Namespace | Size | Access mode | Used by |
| --- | --- | --- | --- | --- |
| `data-pvc` | `karakeep` | `1Gi` | `ReadWriteOnce` | Karakeep web data. |
| `meilisearch-pvc` | `karakeep` | `1Gi` | `ReadWriteOnce` | Meilisearch index data. |
| `recipes-media` | `tandoor` via app destination | `5Gi` | `ReadWriteOnce` | Tandoor media files. |
| `recipes-static` | `tandoor` via app destination | `5Gi` | `ReadWriteOnce` | Tandoor static files. |

Open WebUI also enables chart-managed persistence with a `100Gi` `ReadWriteOnce` volume.

## Notable configuration

- `homepage/kustomization.yaml` generates a `homepage` ConfigMap from files under `homepage/config/`.
- `karakeep/kustomization.yaml` deploys the namespace, web app, Chrome, Meilisearch, services, and PVCs as one unit.
- Tandoor runs migrations and `collectstatic` in an init container before starting Gunicorn behind nginx.
- `argocd/config/argocd-cm.yaml` adds a `homepage` API key account, ignores selected managed fields, and excludes Cilium identities from Argo CD resource tracking.
- `kube-system/ts-dns.yaml` configures Tailscale DNS behavior for the cluster.
