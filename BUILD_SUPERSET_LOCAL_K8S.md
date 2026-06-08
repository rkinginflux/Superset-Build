Superset local K8 build/install runbook (sequential)

Goal
- Build kubectl manifests from the Superset Helm chart in /superset
- Install on local kubeadm cluster
- Run Superset with Postgres metadata DB + Redis, both persistent on local-path PVCs
- Verify Web UI access and data persistence

1) Prerequisites
1.1) Ensure helm binary exists (use native binary, not snap)
  helm version --short

1.2) Ensure kubeconfig is current after cluster rebuild
  sudo cp /etc/kubernetes/admin.conf ~/.kube/admin.conf
  sudo chown $USER:$USER ~/.kube/admin.conf

1.3) Verify cluster access
  kubectl get nodes -o wide

1.4) For single-node kubeadm, remove control-plane taint
  kubectl taint nodes $(hostname) node-role.kubernetes.io/control-plane:NoSchedule-

2) Install local-path StorageClass (Rancher)
2.1) Apply local-path provisioner
  /usr/bin/kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

2.2) Verify storageclass and provisioner pod
  kubectl get storageclass
  kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=180s

Expected: storageclass local-path exists.

3) Use local values profile
- Values file:
  superset/k8s/superset-values-local.yaml

Important local settings in that file:
- service.type: NodePort (32088)
- extraSecretEnv.SUPERSET_SECRET_KEY set
- init.enabled: true
- postgresql.enabled: true
- postgresql.primary.persistence.enabled: true
- postgresql.primary.persistence.storageClass: local-path
- postgresql.primary.persistence.size: 10Gi
- redis.enabled: true
- redis.master.persistence.enabled: true
- redis.master.persistence.storageClass: local-path
- redis.master.persistence.size: 5Gi
- supersetWorker replicaCount: 0
- configOverrides includes ENABLE_PROXY_FIX, WTF_CSRF_ENABLED, ROW_LIMIT

4) Render Helm chart to kubectl YAML
4.1) Build dependencies
  cd /superset
  helm dependency build

4.2) Render manifest
  helm template superset-local . -n superset -f superset-values-local.yaml > superset-local.yaml

5) Install to cluster
5.1) Fresh namespace (recommended for clean retries)
  kubectl delete namespace superset --ignore-not-found=true
  kubectl wait --for=delete namespace/superset --timeout=180s || true
  kubectl create namespace superset

5.2) Apply manifest
  kubectl apply -n superset -f superset-local.yaml

5.3) Wait for init and app rollout
  kubectl wait --for=condition=Ready pod -n superset -l app.kubernetes.io/name=postgresql --timeout=300s
  kubectl wait --for=condition=complete job/superset-local-init-db -n superset --timeout=420s
  kubectl rollout status deployment/superset-local -n superset --timeout=300s

6) Verify persistent volumes
6.1) Check PVCs are Bound
  kubectl get pvc -n superset

Expected:
- data-superset-local-postgresql-0 (Bound, 10Gi, local-path)
- redis-data-superset-local-redis-master-0 (Bound, 5Gi, local-path)

7) Verify service and web access
7.1) Check service
  kubectl get svc -n superset superset-local -o wide

Expected: NodePort 32088 on service port 8088.

7.2) Health check page
  curl -sS -o /dev/null -w '%{http_code}\n' http://192.168.0.30:32088/login/

Expected: 200

7.3) Browser login
- URL: http://192.168.0.30:32088/login/
- Username: admin
- Password: admin

8) Optional persistence smoke checks
8.1) Postgres persistence
- Write a marker row using psycopg2 from superset pod, restart postgres pod, read row again.
- If row remains, PVC persistence is working.

8.2) Redis persistence
- Set a key using redis client from superset pod, restart redis pod, read key again.
- If key remains, PVC persistence is working.

9) Troubleshooting quick map
- Symptom: StatefulSet apply fails with immutable spec error after enabling persistence
  Cause: changing volumeClaimTemplates on existing StatefulSet
  Fix: delete affected StatefulSet (postgres or redis), re-apply manifest

- Symptom: init job fails with ModuleNotFoundError: psycopg2
  Cause: image mismatch for postgres driver
  Fix: ensure bootstrapScript installs psycopg2-binary (already included in local values)

- Symptom: kubectl TLS x509 unknown authority
  Fix: refresh /home/dad/.kube/admin.conf from /etc/kubernetes/admin.conf

- Symptom: pods Pending on single-node control-plane
  Fix: remove control-plane NoSchedule taint

10) Useful files
- superset-values-local.yaml
- superset-local.yaml
- superset-default.yaml
- superset-docs-aligned.yaml
