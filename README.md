
# Vault + External Secrets Operator (ESO) Setup on Kubernetes

Panduan lengkap deploy HashiCorp Vault (dev mode), External Secrets Operator (ESO), dan contoh aplikasi yang menggunakan secret dari Vault via ESO di Kubernetes.

---

## 1. Deploy Vault (Dev Mode)

Buat file `vault/values.yaml`:

```yaml
server:
  dev:
    enabled: true
  dataStorage:
    enabled: false
  extraEnvironmentVars:
    VAULT_DEV_ROOT_TOKEN_ID: "root"
```

Install Vault dengan Helm:

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault --create-namespace \
  -f vault/values.yaml
```

Cek pod Vault:

```bash
kubectl get pods -n vault
```

---

## 2. Konfigurasi Vault

Masuk ke pod Vault (ganti nama pod sesuai):

```bash
kubectl exec -it pod/vault-0 -n vault -- sh
```

Set environment variables:

```bash
export VAULT_TOKEN=root
export VAULT_ADDR=http://127.0.0.1:8200
vault status
```

### 2.1 Enable KV v2 dan Tambah Secret Contoh

```bash
vault secrets enable -path=secret kv-v2
vault kv put secret/myapp username="admin" password="s3cr3t"
```

### 2.2 Enable Kubernetes Auth dan Konfigurasi

```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://${KUBERNETES_PORT_443_TCP_ADDR}:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### 2.3 Buat Policy dan Role untuk ESO

Buat policy `read-secrets`:

```bash
vault policy write read-secrets - <<EOF
path "secret/data/*" {
  capabilities = ["read"]
}
EOF
```

Buat role untuk ServiceAccount `external-secrets`:

```bash
vault write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=read-secrets \
  ttl=24h
```

(Opsi) Role untuk app yang akses Vault langsung:

```bash
vault write auth/kubernetes/role/myapp-role \
  bound_service_account_names=default \
  bound_service_account_namespaces=default \
  policies=read-secrets \
  ttl=24h
```

Keluar dari pod Vault:

```bash
exit
```

---

## 3. Install External Secrets Operator (ESO)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true
```

Cek pod:

```bash
kubectl get pods -n external-secrets
```

---

## 4. Buat ClusterSecretStore (ESO → Vault)

Buat file `external-secrets/clustersecretstore-vault.yaml`:

```yaml
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
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
```

Apply dan cek status:

```bash
kubectl apply -f external-secrets/clustersecretstore-vault.yaml
kubectl get clustersecretstore vault-backend -o yaml
```

Pastikan status `READY` `True`.

---

## 5. Buat ExternalSecret (Sinkron Vault → K8s Secret)

Buat file `external-secrets/externalsecret-myapp.yaml`:

```yaml
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
```

Apply dan cek:

```bash
kubectl apply -f external-secrets/externalsecret-myapp.yaml
kubectl get externalsecret -n default
kubectl describe externalsecret myapp-secret -n default
kubectl get secret myapp-secret -o yaml
```

---

## 6. Deploy Aplikasi Sample dengan Secret dari Vault

Buat file `app/deployment.yaml`:

```yaml
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
```

Apply dan cek pod:

```bash
kubectl apply -f app/deployment.yaml
kubectl get pods -n default -w
```

Cek environment variables di pod:

```bash
kubectl exec -it deploy/myapp -n default -- sh -c 'env | grep -E "(username|password)" || true'
```

---

## Catatan

- Vault berjalan dalam mode **dev**, tidak untuk produksi.
- Pastikan nama namespace dan service account sesuai konfigurasi Vault dan ESO.
- Jika secrets tidak sinkron, cek log operator dan konfigurasi role Vault.
- Sesuaikan `refreshInterval` di ExternalSecret jika perlu.

