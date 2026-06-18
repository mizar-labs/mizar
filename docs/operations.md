# Operations guide

This repository is intended to be operated through GitOps. Treat Git as the source of truth and let Argo CD converge the cluster.

## Bootstrap sequence

Use this flow for a new cluster or a recovery where Argo CD is not yet running:

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f argocd/install.yaml
kubectl apply -f argocd/config/
kubectl apply -f apps/apps.yaml
```

Once `apps/apps.yaml` is applied, Argo CD should create the rest of the applications declared in `apps/`.

## Inspecting sync state

With the Argo CD CLI:

```bash
argocd app list
argocd app get apps
argocd app get <app-name>
argocd app diff <app-name>
argocd app sync <app-name>
```

With kubectl only:

```bash
kubectl -n argocd get applications.argoproj.io
kubectl -n argocd describe application <app-name>
kubectl -n argocd get pods
```

## Validating changes locally

At minimum, parse the Kubernetes YAML before committing. Skip `homepage/config/*.yaml`: those files are packaged as raw ConfigMap data by kustomize, and may contain unquoted Homepage template variables such as `{{HOMEPAGE_VAR_ICAL_URL}}` that are valid for the application but not valid standalone YAML for PyYAML.

```bash
python3 - <<'PY'
from pathlib import Path
import yaml
for path in Path('.').rglob('*.yaml'):
    if path.parts[:2] == ('homepage', 'config'):
        continue
    try:
        list(yaml.safe_load_all(path.read_text()))
    except Exception as exc:
        raise SystemExit(f'{path}: {exc}')
print('all YAML parsed')
PY
```

For kustomize apps:

```bash
kubectl kustomize homepage
kubectl kustomize karakeep
```

For cluster-side schema validation, run a server dry-run against the target cluster:

```bash
kubectl apply --dry-run=server -f <manifest.yaml>
```

Use server-side validation for CRDs such as `Application`, `IngressRoute`, `Certificate`, and `DNSConfig`, because local Kubernetes validators will not know those schemas unless configured with the cluster CRDs.

## Adding a new application

1. Create a new top-level directory, for example `my-service/`.
2. Add Kubernetes manifests for the workload, services, config, storage, and namespace if needed.
3. If multiple files are involved, add a `kustomization.yaml` so the directory can be rendered as one unit.
4. Add `apps/my-service.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mizar-labs/mizar
    path: my-service
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: my-service
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

5. Add an ingress manifest under `ingress/` if the service needs an external route.
6. Commit and push. Argo CD will reconcile it through the `apps` app.

## Updating an existing application

1. Edit manifests in the owning top-level directory.
2. Keep names, labels, selectors, service ports, and ingress backends consistent.
3. Render kustomize directories after changes.
4. Commit and push.
5. Watch Argo CD and workload rollout status.

Useful rollout commands:

```bash
kubectl -n <namespace> rollout status deployment/<name>
kubectl -n <namespace> rollout status statefulset/<name>
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs deploy/<name> --all-containers --tail=200
```

## Secrets

The repo references secrets that are expected to exist in the cluster. Do not add secret values directly to Git.

Current known secret references include:

| Secret | Referenced by | Purpose |
| --- | --- | --- |
| `argocd-redis` | Argo CD Redis/server/repo/controller manifests | Redis authentication. |
| `recipes` | Tandoor deployment and PostgreSQL stateful set | Django secret key, PostgreSQL password, email settings, S3 settings, AI settings. |

Recommended approaches:

- Use External Secrets Operator, Sealed Secrets, SOPS, or another encrypted-secret workflow.
- Ensure secrets are created before enabling automated sync for dependent apps.
- Keep secret key names stable; deployments reference individual keys explicitly.

## Ingress conventions

Traefik-backed external routes in this repo generally use:

- `kind: IngressRoute`
- `entryPoints: [websecure]`
- a matching cert-manager `Certificate`
- hosts under `mizar.scalar.cloud`

When adding a route, check all of these match:

- route host in `spec.routes[].match`
- certificate `spec.dnsNames[]`
- route TLS `secretName`
- certificate `spec.secretName`
- backend service name, namespace, and port

## Storage conventions

Persistent apps use PVCs checked into Git unless the Helm chart creates the claim. Existing PVCs should be treated carefully:

- Do not rename a PVC unless you have a migration plan.
- Avoid changing access modes in-place.
- Increasing size is usually safe if the storage class supports expansion; decreasing size is not.
- Tandoor mounts media and static PVCs with `subPath: files` to avoid ext4 `lost+found` issues.

## Troubleshooting quick paths

### App is missing

```bash
kubectl -n argocd get application apps -o yaml
kubectl -n argocd describe application apps
```

Check whether the app manifest exists under `apps/` and whether the root `apps` app is synced.

### App is out of sync

```bash
argocd app diff <app-name>
argocd app sync <app-name>
```

If Argo CD reports unknown kinds, verify the relevant CRD is installed first.

### Pods are pending

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get pvc
```

Common causes are missing PVC binding, missing secrets, or insufficient cluster resources.

### Pods are crashing

```bash
kubectl -n <namespace> logs <pod> --all-containers --previous
kubectl -n <namespace> describe pod <pod>
```

For Tandoor, check the init container first because migrations and static collection run before the main containers start.

### Ingress returns TLS or 404 errors

```bash
kubectl -n <namespace> get ingress,ingressroute,certificate,secret
kubectl -n <namespace> describe certificate <name>
kubectl -n <namespace> describe ingressroute <name>
```

Verify the DNS name, certificate secret, Traefik entry point, and backend service port.
