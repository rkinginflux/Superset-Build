Here is the exact sequential install flow (copy/paste):

1) Make sure local-path storage exists (safe to re-run)
```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=180s
kubectl get storageclass
```
2) Create namespace
```
kubectl create namespace superset --dry-run=client -o yaml | kubectl --kubeconfig=~/.kube/admin.conf apply -f -
```
3) Apply Superset manifest
```
kubectl apply -n superset -f superset-local.yaml
```
4) Wait for Postgres + Redis + Superset
```
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=postgresql --timeout=300s
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=redis --timeout=300s
kubectl rollout status deployment/superset-local -n superset --timeout=300s
```
5) Verify PVCs (persistence)
```
kubectl get pvc -n superset
```
6) Verify web access
```
kubectl get svc -n superset superset-local
curl -I http://192.168.0.30:32088/login/
```
If you edit values files later:
- values files are NOT directly applied with kubectl.
- - you must re-render manifest with helm, then apply the new rendered yaml.
