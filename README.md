# Superset-Build
Superset Helm -> kubectl manifest exports

Files:
- superset/k8s/manifests/superset-default.yaml
  - Rendered with chart defaults
  - Command:
    helm template superset-default superset/helm/superset -n superset

- superset/k8s/manifests/superset-local.yaml
  - Rendered with local override values
  - Values file: superset/k8s/superset-values-local.yaml
  - Command:
    helm template superset-local superset/helm/superset -n superset -f superset/k8s/superset-values-local.yaml
  - Current local profile summary:
    - Superset metadata DB: Postgres (chart dependency)
    - Postgres persistence: enabled (PVC, storageClass local-path, size 10Gi)
    - Redis persistence: enabled (PVC, storageClass local-path, size 5Gi)
    - Service exposure: NodePort 32088

Apply examples:
- kubectl create namespace superset
- kubectl apply -n superset -f superset/k8s/manifests/superset-local.yaml

PVC verification:
- kubectl get pvc -n superset
