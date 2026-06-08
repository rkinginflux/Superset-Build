Sequential install (copy/paste)

Assumes:
- You already have kubectl context working (or `$KUBECONFIG` set)
- You are in this repo directory (`Superset-Build`)

1) Install local-path storage (safe to re-run)
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=180s
kubectl get storageclass
```

2) Create namespace
```bash
kubectl create namespace superset --dry-run=client -o yaml | kubectl apply -f -
```

3) Apply Superset manifest
```bash
kubectl apply -n superset -f manifest/superset-local.yaml
```

4) Wait for workloads
```bash
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=postgresql --timeout=300s
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=redis --timeout=300s
kubectl rollout status deployment/superset-local -n superset --timeout=300s
```

5) Verify persistence
```bash
kubectl get pvc -n superset
```
Expected bound claims:
- `data-superset-local-postgresql-0`
- `redis-data-superset-local-redis-master-0`

6) Verify web access
```bash
kubectl get svc -n superset superset-local
curl -I http://192.168.0.30:32088/login/
```

Note if you edit values files later
- Values are inputs to Helm templates.
- Re-render manifest with Helm and apply the rendered YAML.
