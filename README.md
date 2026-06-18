# mizar

Mizar is a GitOps repository for a Kubernetes homelab cluster. Argo CD reconciles this repository into the cluster using an app-of-apps layout: the root `apps` application points at `apps/`, and each file in `apps/` defines an Argo CD `Application` for one workload or platform area.

## What is in this repo

| Path | Purpose |
| --- | --- |
| `apps/` | Argo CD `Application` manifests. This is the entry point for GitOps reconciliation. |
| `argocd/` | In-cluster Argo CD install manifest plus local Argo CD configuration. |
| `ingress/` | Shared public/private ingress resources, mostly Traefik `IngressRoute` plus cert-manager `Certificate` objects. |
| `kube-system/` | Cluster-level networking helpers such as Tailscale DNS/egress services and Hubble UI ingress. |
| `homepage/` | [Homepage](https://gethomepage.dev/) dashboard deployment and generated config maps. |
| `karakeep/` | Karakeep stack: web app, Meilisearch, Chrome sidecar service, namespace, PVCs. |
| `tandoor/` | Tandoor Recipes stack: web/nginx deployment, PostgreSQL stateful set, PVCs, service account, config. |
| `glance/` | Glance dashboard deployment and service. |
| `foundryvtt/` | Foundry VTT deployment and service. |
| `epic-free-games*/` | Epic free-games notifier/claimer workloads for two namespaces. |
| `free-games-claimer*/` | Free Games Claimer workloads for two namespaces. |

See [`docs/app-catalog.md`](docs/app-catalog.md) for a service-by-service inventory and [`docs/operations.md`](docs/operations.md) for common operational workflows.

## GitOps model

1. Bootstrap Argo CD into the cluster from `argocd/`.
2. Apply `apps/apps.yaml` once.
3. Argo CD discovers the other manifests in `apps/` and continuously reconciles each application.

All local applications use:

- `repoURL: https://github.com/mizar-labs/mizar`
- `targetRevision: HEAD`
- automated sync with `prune: true` and `selfHeal: true`
- `CreateNamespace=true` where configured

The `open-webui` application is the exception: it is sourced from the Open WebUI Helm repository instead of this Git repository.

## Bootstrap

Prerequisites:

- Kubernetes cluster access through `kubectl`
- Argo CD CRDs/controllers installed from this repository or otherwise available
- Traefik CRDs for `IngressRoute` manifests
- cert-manager CRDs for `Certificate` manifests
- Tailscale operator CRDs for `DNSConfig`

Initial install from a checked-out copy:

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f argocd/install.yaml
kubectl apply -f argocd/config/
kubectl apply -f apps/apps.yaml
```

After that, Argo CD owns the desired state. Prefer changing YAML in Git over editing live cluster objects.

## Day-to-day workflow

Add or change a workload:

1. Create or edit the workload directory.
2. If it should be managed by Argo CD, add or update its `apps/<name>.yaml` application manifest.
3. Add ingress in `ingress/` when the service needs external routing.
4. Commit and push.
5. Watch Argo CD reconcile the application.

Useful checks before committing:

```bash
# Parse Kubernetes YAML files. Skip Homepage config snippets because
# kustomize packages them as raw ConfigMap data and they may contain
# unquoted Homepage template variables such as {{HOMEPAGE_VAR_ICAL_URL}}.
python3 - <<'PY'
from pathlib import Path
import yaml
for path in Path('.').rglob('*.yaml'):
    if path.parts[:2] == ('homepage', 'config'):
        continue
    list(yaml.safe_load_all(path.read_text()))
    print(path)
PY

# Render kustomize-based apps
kubectl kustomize homepage
kubectl kustomize karakeep
```

## Secrets and configuration

Some manifests reference Kubernetes `Secret` objects that are not defined in this repository. Those secrets must exist in-cluster before the corresponding pods can become ready. Examples include:

- `recipes` in the Tandoor namespace for database, email, S3, and AI settings
- `argocd-redis` for Argo CD Redis auth
- application-specific environment variables consumed by Homepage widgets

Do not commit real secret values to this repository. Use an external secret manager, sealed/encrypted secrets, or pre-created cluster secrets.

## Documentation

- [`docs/app-catalog.md`](docs/app-catalog.md) — inventory of Argo CD apps, workloads, images, ingress hosts, and storage.
- [`docs/operations.md`](docs/operations.md) — bootstrap, validation, rollout, troubleshooting, and conventions.
