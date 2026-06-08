# Superset-Build

Pre-rendered Kubernetes manifests and values files for deploying Apache Superset.

Repo layout
- `manifest/superset-local.yaml` - local profile manifest (NodePort 32088, Postgres + Redis)
- `manifest/superset-default.yaml` - chart defaults manifest
- `manifest/superset-docs-aligned.yaml` - docs-aligned variant manifest
- `superset-values-local.yaml` - local values profile (reference)
- `superset-values-prod.yaml` - production template values (reference)
- `superset-values-docs-aligned.yaml` - docs-aligned values (reference)
- `Install.md` - quickest install path
- `BUILD_SUPERSET_LOCAL_K8S.md` - detailed local runbook
- `BUILD_SUPERSET_PROD_STYLE_K8S.md` - production-style runbook

Quick install (pre-rendered manifest)
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=180s
kubectl create namespace superset --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n superset -f manifest/superset-local.yaml
kubectl rollout status deployment/superset-local -n superset --timeout=300s
```

Verify
```bash
kubectl get pods -n superset
kubectl get pvc -n superset
kubectl get svc -n superset superset-local
curl -I http://192.168.0.30:32088/login/
```

Important
- You can rely on `$KUBECONFIG`; no `--kubeconfig` flag is required if your env var is set.
- `~` paths are shell-expanded and are fine in docs/scripts.
- Values files are not applied directly with kubectl. If you change values, re-render a manifest with Helm, then apply that rendered YAML.
