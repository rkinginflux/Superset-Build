# Superset-Build

Pre-rendered Kubernetes manifests and values files for deploying Apache Superset.

Repo layout
- `manifest/superset-local.yaml` - local profile manifest (NodePort 32088, Postgres + Redis)
- `manifest/superset-default.yaml` - chart defaults manifest
- `manifest/superset-docs-aligned.yaml` - docs-aligned variant manifest
- `superset-values-local.yaml` - local values profile (reference)
- `superset-values-prod.yaml` - production template values (reference)
- `superset-values-docs-aligned.yaml` - docs-aligned values (reference)
- `BUILD_SUPERSET_LOCAL_K8S.md` - detailed local runbook
- `BUILD_SUPERSET_PROD_STYLE_K8S.md` - production-style runbook

Quick install (pre-rendered manifest)
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=180s
kubectl create namespace superset --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n superset -f manifest/superset-local.yaml
kubectl rollout status deployment/superset-local -n superset --timeout=300s
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=postgresql --timeout=300s
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=redis --timeout=300s
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

Lessons Learned (Superset + InfluxDB 3 Enterprise lab)
- FlightSQL driver in Superset: if database test fails with `Could not load database driver: BaseEngineSpec` for `datafusion+flightsql://...`, install `flightsql-dbapi` in the Superset runtime.
- Transport mode matters: for in-cluster `:8181` endpoints serving non-TLS Flight/gRPC (h2c), add `insecure=true` to the SQLAlchemy URI, or TLS handshake fails (`SSL_ERROR_SSL: wrong version number`).
- Namespace parameter is required: include a namespace header via URI parameter (for example `bucket-name=<name>`), otherwise FlightSQL requests fail with missing database/namespace context.
- `INFLUXDB3_DISABLE_AUTHZ` scope is limited to `health`, `metrics`, and `ping` endpoints. It does not disable authz for query/FlightSQL APIs.
- Offline token bootstrap in Kubernetes: use Secret `--from-file` for token JSON artifacts. `--from-literal` only stores strings and does not persist token JSON content.
- Mount token files into the pod and point env vars to mounted paths:
  - `INFLUXDB3_ADMIN_TOKEN_FILE=/plugins/admin-token-offline.json`
  - `INFLUXDB3_PERMISSION_TOKENS_FILE=/plugins/permission-tokens-offline.json`
- Permission token bootstrap can pre-create lab namespaces/databases using `create_databases` in the permission token file.
- Current observed caveat: HTTP SQL API worked with bootstrapped permission tokens in this lab, while FlightSQL still returned an authz verification error for schema-related actions. Treat this as a server-side authz behavior to validate per version/build.
