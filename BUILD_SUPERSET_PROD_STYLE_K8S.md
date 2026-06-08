Superset production-style K8 runbook (sequential)

Goal
- Deploy Superset on Kubernetes with production-oriented defaults
- Use PostgreSQL metadata DB (not sqlite)
- Keep configuration maintainable via values + rendered manifests

Assumptions
- `kubectl` points to your cluster (or `$KUBECONFIG` is set)
- Helm is installed where you run render commands
- You have access to a Superset Helm chart checkout for rendering

Storage profile guidance (important)
- Local/dev single-node: `local-path` is fine.
- Production/multi-node: use a shared, durable CSI storage class.
  Examples: Longhorn, Rook/Ceph RBD, EBS/GCE PD/Azure Disk classes.
- Do not treat `local-path` as HA storage.

1) Prerequisites
1.1) Verify tools
```bash
helm version --short
kubectl version --client
kubectl get nodes -o wide
```

1.2) For single-node kubeadm only, remove control-plane taint
```bash
kubectl taint nodes $(hostname) node-role.kubernetes.io/control-plane:NoSchedule-
```

2) Create hardened prod values
- Start from: `superset-values-prod.yaml`
- Replace all `REPLACE_*` values
- Set `storageClass` explicitly for Postgres/Redis persistence

2.1) Must-change security values
- `SUPERSET_SECRET_KEY` (strong random)
- Postgres password(s)
- Redis password
- Admin password

3) Render prod manifest (from chart checkout)
Example (adjust `<chart-path>`):
```bash
helm dependency build <chart-path>
helm template superset-prod <chart-path> -n superset -f superset-values-prod.yaml > manifest/superset-prod.yaml
```

4) Install/upgrade in cluster
```bash
kubectl create namespace superset --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n superset -f manifest/superset-prod.yaml
```

5) Wait and verify
```bash
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=postgresql --timeout=600s
kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=redis --timeout=600s
kubectl wait --for=condition=complete job/superset-prod-init-db -n superset --timeout=900s
kubectl wait --for=condition=Available deployment/superset-prod -n superset --timeout=900s
kubectl get pvc -n superset
kubectl get pods,svc,ingress -n superset
```

6) Validate web access
- Ingress mode:
```bash
curl -k -I https://superset.example.com/health
```
- NodePort fallback:
```bash
curl -I http://<node-ip>:<nodeport>/health
```

7) Post-install hardening
- Rotate admin password
- Restrict network access (NetworkPolicy)
- Externalize secrets (Sealed Secrets / External Secrets)
- Back up metadata DB PVCs or use managed Postgres
- Configure SMTP/OAuth/SSO as needed

8) Troubleshooting
- Init job fails with missing postgres driver:
  - verify image/bootstrap includes required postgres client deps.
- Login loops behind ingress:
  - verify proxy headers and Superset proxy settings.
- StatefulSet immutable errors after persistence template changes:
  - recreate affected StatefulSet and re-apply manifest.

9) Artifacts in this repo
- `superset-values-prod.yaml` (template)
- `manifest/superset-default.yaml`
- `manifest/superset-local.yaml`
- `manifest/superset-docs-aligned.yaml`
- `Install.md`
