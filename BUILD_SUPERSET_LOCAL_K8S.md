Superset local K8 install runbook (sequential)

Goal
- Deploy Superset to a local kubeadm cluster
- Use pre-rendered manifest in this repo
- Run with Postgres + Redis persistence on local-path

Assumptions
- `kubectl` points to your target cluster (or `$KUBECONFIG` is set)
- You are in this repo root (`Superset-Build`)

1) Prerequisites
1.1) Verify cluster access
```bash
kubectl get nodes -o wide
```

1.2) For single-node kubeadm, remove control-plane taint (if needed)
```bash
kubectl taint nodes $(hostname) node-role.kubernetes.io/control-plane:NoSchedule-
```
(If already removed, kubectl will print an expected "not found"/"already" style message.)

2) Install local-path StorageClass
2.1) Apply provisioner
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

2.2) Verify
```bash
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=180s
kubectl get storageclass
```
Expected: `local-path` exists.

3) Install Superset from manifest
3.1) Create namespace
```bash
kubectl create namespace superset --dry-run=client -o yaml | kubectl apply -f -
```

3.2) Apply local manifest
```bash
kubectl apply -n superset -f manifest/superset-local.yaml
```

3.3) Wait for readiness
```bash
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=postgresql --timeout=300s
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=redis --timeout=300s
kubectl rollout status deployment/superset-local -n superset --timeout=300s
```

4) Verify persistence
```bash
kubectl get pvc -n superset
```
Expected Bound claims:
- `data-superset-local-postgresql-0` (10Gi)
- `redis-data-superset-local-redis-master-0` (5Gi)

5) Verify web access
5.1) Check service
```bash
kubectl get svc -n superset superset-local -o wide
```
Expected: NodePort `32088` on port `8088`.

5.2) Health/login page
```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://192.168.0.30:32088/login/
```
Expected: `200`

5.3) Browser login
- URL: `http://192.168.0.30:32088/login/`
- Username: `admin`
- Password: `admin`

6) If you need to re-render manifests from values
This repo ships pre-rendered manifests. Values files are for reference/editing.
If you change values files, render a new manifest from a Superset chart checkout, then apply that rendered YAML.

7) Troubleshooting quick map
- StatefulSet immutable field error after storage changes:
  - delete affected StatefulSet (postgres/redis), then re-apply manifest.
- Pods Pending on single-node:
  - re-check control-plane taint removal and node resources.
- Login/API errors:
  - inspect deployment and init-job logs first.

8) Repo artifacts used in this flow
- `manifest/superset-local.yaml`
- `superset-values-local.yaml`
- `Install.md`
