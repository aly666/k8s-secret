1 — Deploy Vault (Helm, dev mode)

Buat file vault/values.yaml:
server:
  dev:
    enabled: true
  dataStorage:
    enabled: false
  extraEnvironmentVars:
    VAULT_DEV_ROOT_TOKEN_ID: "root"

Install:
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault --create-namespace \
  -f vault/values.yaml

Cek pod:
kubectl get pods -n vault
# nanti biasanya nama pod: vault-0  atau deploy/vault tergantung chart version

2 — Konfigurasi Vault (jalankan dalam pod Vault agar mudah)

Masuk ke pod Vault:

# ganti nama pod sesuai output `kubectl get pods -n vault`
kubectl exec -it pod/vault-0 -n vault -- sh
# atau kubectl exec -it vault-0 -n vault -- sh

Di dalam shell pod:
export VAULT_TOKEN=root
export VAULT_ADDR=http://127.0.0.1:8200

# cek status
vault status

2.1 Enable KV v2 & tambah secret contoh
vault secrets enable -path=secret kv-v2
vault kv put secret/myapp username="admin" password="s3cr3t"

2.2 Enable kubernetes auth & konfigurasi koneksi ke API server

(Ini memakai serviceaccount token present dalam pod)
vault auth enable kubernetes

vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://${KUBERNETES_PORT_443_TCP_ADDR}:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

2.3 Buat policy dan role untuk ESO (External Secrets)

Policy (read-secrets) — beri akses baca ke path secret/data/*:

vault policy write read-secrets - <<EOF
path "secret/data/*" {
  capabilities = ["read"]
}
EOF

Buat role yang mengikat ke ServiceAccount external-secrets di namespace external-secrets:

vault write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=read-secrets \
  ttl=24h

2.4 (Opsional) Buat role app untuk testing app langsung
Jika kamu mau app mengakses Vault langsung sendiri (bukan via ESO):
vault write auth/kubernetes/role/myapp-role \
  bound_service_account_names=default \
  bound_service_account_namespaces=default \
  policies=read-secrets \
  ttl=24h

Keluar dari pod Vault:
exit

3 — Install External Secrets Operator (Helm) dengan CRDs

helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true

cek pod:
kubectl get pods -n external-secrets

4 — Buat ClusterSecretStore (ESO → Vault)

File external-secrets/clustersecretstore-vault.yaml:
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      # perhatikan mountPath yang benar: "kubernetes"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"

Apply:
kubectl apply -f external-secrets/clustersecretstore-vault.yaml
Cek status store:
kubectl get clustersecretstore vault-backend -o yaml
# atau
kubectl get clustersecretstore
Store harus READY True (jika belum, cek logs operator).

5 — Buat ExternalSecret (sinkron secret dari Vault → K8s Secret)

File external-secrets/externalsecret-myapp.yaml (gunakan apiVersion v1 sesuai CRD yang ada):
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: myapp-secret
  namespace: default
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: myapp-secret
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: myapp
        property: username
    - secretKey: password
      remoteRef:
        key: myapp
        property: password


Catatan: remoteRef.key gunakan kunci sesuai path Vault.
Untuk KV v2, ESO biasanya mendukung key: myapp (karena kita enabled secret path and wrote to secret/myapp earlier). 
Jika tidak cocok, gunakan key: secret/myapp tergantung behavior chart — tapi percobaan umum: key: myapp dengan path: secret di ClusterSecretStore.

Apply:
kubectl apply -f external-secrets/externalsecret-myapp.yaml
Cek status ExternalSecret:
kubectl get externalsecret -n default
kubectl describe externalsecret myapp-secret -n default
Setelah Ready: True, secret Kubernetes akan muncul:
kubectl get secret myapp-secret -o yaml
Jika belum muncul, cek logs controller:
kubectl logs -n external-secrets deploy/external-secrets
Pesan umum error & perbaikan:
ClusterSecretStore "vault-backend" not found → pastikan apply store.
permission denied saat login ke Vault → pastikan role & token reviewer config di Vault dan mountPath benar.

6 — Deploy sampel App yang pakai secret (envFrom)

File app/deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: nginx
          envFrom:
            - secretRef:
                name: myapp-secret

Apply:
kubectl apply -f app/deployment.yaml
kubectl get pods -w

# k8s-secret
