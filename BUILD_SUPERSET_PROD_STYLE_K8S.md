Superset production-style K8 runbook (sequential)

Goal
- Deploy Superset on Kubernetes with production-oriented defaults
- Use PostgreSQL metadata DB (not sqlite)
- Keep config manageable via Helm values + overrides

Assumptions
- Repo: ~/superset
- Kubeconfig: ~/.kube/admin.conf
- Helm binary: ~/.local/bin/helm
- Namespace: superset

Storage profile guidance (important)
- Local/dev single-node (your current kubeadm lab): use Rancher local-path StorageClass.
- Production/multi-node: use a durable shared CSI-backed StorageClass (examples: Longhorn, Rook/Ceph RBD, EBS/GCE PD/Azure Disk).
- Do not treat local-path as HA storage. It is node-local and suited for local labs.

1) Prerequisites
1.1) Verify tools
  ~/.local/bin/helm version --short
  kubectl  version --client

1.2) Verify cluster
  kubectl  get nodes -o wide

1.3) For single-node kubeadm only, remove control-plane taint
  kubectl  taint nodes poundcake node-role.kubernetes.io/control-plane:NoSchedule-

2) Create production values file
- Start from this local template and harden:
  ~/superset/k8s/superset-values-docs-aligned.yaml

2.1) Must-change security values
- SUPERSET_SECRET_KEY: strong random
  openssl rand -base64 42
- Postgres passwords: strong, non-default
- Admin password: strong, non-default

2.2) Recommended baseline settings
- Use direct image repo:
  image.repository: apache/superset
- Keep Postgres enabled for metadata DB:
  postgresql.enabled: true
- Keep Redis enabled:
  redis.enabled: true
- Add resource requests/limits for supersetNode/postgresql/redis
- Enable persistence for PostgreSQL and Redis
- Explicitly set storageClass for persistence:
  - local-path for local/dev single-node
  - your shared CSI StorageClass for production
- Prefer ingress + TLS over NodePort for external access

3) Example production values skeleton
Save as:
- ~/superset/k8s/superset-values-prod.yaml

Suggested structure:

image:
  repository: apache/superset

extraSecretEnv:
  SUPERSET_SECRET_KEY: "REPLACE_WITH_STRONG_RANDOM"

service:
  type: ClusterIP

ingress:
  enabled: true
  ingressClassName: nginx
  hosts:
    - superset.example.com
  path: /
  pathType: Prefix
  tls:
    - secretName: superset-tls
      hosts:
        - superset.example.com

configOverrides:
  production: |
    ENABLE_PROXY_FIX = True
    WTF_CSRF_ENABLED = True
    ROW_LIMIT = 5000

bootstrapScript: |
  #!/bin/bash
  uv pip install .[postgres]
  if [ ! -f ~/bootstrap ]; then
    echo "Running Superset with uid {{ .Values.runAsUser }}" > ~/bootstrap
  fi

init:
  adminUser:
    username: admin
    firstname: Superset
    lastname: Admin
    email: admin@superset.local
    password: "REPLACE_WITH_STRONG_PASSWORD"

postgresql:
  enabled: true
  auth:
    username: superset
    password: "REPLACE_WITH_STRONG_PASSWORD"
    database: superset
  primary:
    persistence:
      enabled: true
      storageClass: "REPLACE_WITH_STORAGECLASS"  # local-path (local/dev) or shared CSI class (prod)
      size: 20Gi

redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    password: "REPLACE_WITH_STRONG_PASSWORD"
  master:
    persistence:
      enabled: true
      storageClass: "REPLACE_WITH_STORAGECLASS"  # local-path (local/dev) or shared CSI class (prod)
      size: 8Gi

supersetNode:
  replicas:
    enabled: true
    replicaCount: 2
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2
      memory: 4Gi

supersetWorker:
  replicas:
    enabled: true
    replicaCount: 1
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2
      memory: 4Gi

4) Render Helm chart to kubectl YAML
4.1) Build chart deps
  cd ~/superset/helm/superset
  ~/.local/bin/helm dependency build

4.2) Render prod manifest
  ~/.local/bin/helm template superset-prod . -n superset \
    -f ~/superset/k8s/superset-values-prod.yaml \
    > ~/superset/k8s/manifests/superset-prod.yaml

5) Install/upgrade in cluster
5.1) Create namespace if needed
  kubectl  create namespace superset --dry-run=client -o yaml | \
    kubectl  apply -f -

5.2) Apply manifest
  kubectl  apply -n superset -f ~/superset/k8s/manifests/superset-prod.yaml

5.3) Wait for readiness
  kubectl  wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=postgresql --timeout=600s
  kubectl  wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=redis --timeout=600s
  kubectl  wait --for=condition=complete job/superset-prod-init-db -n superset --timeout=900s
  kubectl  wait --for=condition=Available deployment/superset-prod -n superset --timeout=900s

5.4) Verify PVCs are bound (when persistence enabled)
  kubectl  get pvc -n superset

6) Validate installation
6.1) Objects
  kubectl  get pods,svc,ingress -n superset

6.2) Web health
- Ingress mode:
  curl -k -I https://superset.example.com/health
- NodePort fallback:
  curl -I http://<node-ip>:<nodeport>/health

6.3) Login
- URL: https://superset.example.com/login/
- User: configured admin user
- Password: configured admin password

7) Post-install hardening
- Rotate admin password immediately
- Replace example hostnames/certs
- Restrict network access (NetworkPolicy)
- Externalize secrets (Sealed Secrets / External Secrets)
- Back up Postgres PVC or move metadata DB to managed Postgres
- Configure SMTP/OAuth/SSO if required

8) Troubleshooting map
- If init job fails with psycopg2 missing:
  confirm bootstrapScript installs postgres extras and image supports runtime install
- If login loops:
  verify ENABLE_PROXY_FIX and forwarded headers at ingress
- If 500 after login:
  check web pod logs and init job logs first
- If pods Pending on single-node:
  re-check control-plane taint and resources

9) Artifacts
- Values: ~/superset/k8s/superset-values-prod.yaml
- Manifest: ~/superset/k8s/manifests/superset-prod.yaml
- This runbook: ~/superset/k8s/BUILD_SUPERSET_PROD_STYLE_K8S.md
