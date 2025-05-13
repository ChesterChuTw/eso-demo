# Core Concept

- 以 Vault Token 為認證方式拉取 Secrets

# 🧠 架構流程簡述

- OpenBao 建立一筆 KV Secret（例如帳密）
- 建立一組 Vault Token（通常為 root token 或 policy 限制後的 token）
- 透過 ESO 建立 `SecretStore`（指定 Vault Server、token 認證方式）
- 建立 `ExternalSecret` 物件，同步指定路徑的 Vault Secrets 到 Kubernetes Secret
- 檢查是否同步成功，並可觀察 logs 或直接 decode Secret 值驗證

# OpenBao

## **Prerequisite**

```bash
bao secrets enable -path=secret kv-v2
```

## Flow

```bash
NAMESPACE=demo1
VAULT_SERVER_ENDPOINT="http://192.168.1.240:8200"

# 建立 DEMO DATA
bao kv put secret/demo-secret-1 username=demo-user-1 password=demo-basic-usage

###############################################################################

kubectl create ns $NAMESPACE

# 建立 Vault Token Secret
cat <<EOF > vault-token.yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: $NAMESPACE
data:
  token: cy5xQjhuQXpOWGdvMmdaU1VBMkRzUkNrb3Q= # root token
EOF

kubectl apply -f vault-token.yaml

# 建立 SecretStore (指定驗證方式)
cat <<EOF > secretstore.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: demo-ss-1
  namespace: $NAMESPACE
spec:
  provider:
    vault:
      server: "$VAULT_SERVER_ENDPOINT"
      path: "secret"
      # Version is the Vault KV secret engine version.
      # This can be either "v1" or "v2", defaults to "v2"
      version: "v2"
      auth:
        # points to a secret that contains a vault token
        # https://www.vaultproject.io/docs/auth/token
        tokenSecretRef:
          name: "vault-token"
          key: "token"
EOF

kubectl apply -f secretstore.yaml

# 建立 ExternalSecret (指定要同步的資料)
cat <<EOF > externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: demo-es-1
  namespace: $NAMESPACE
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: demo-ss-1
    kind: SecretStore
  target:
    name: sync-secret-1
  data:
    - secretKey: demo-1-username # 對應產生 secret (sync-secret-1) 的 key
      remoteRef:
        key: demo-secret-1 # 對應 Vault 的 secret 路徑: secret/data/demo-secret-1
        property: username
    - secretKey: demo-1-password
      remoteRef:
        key: demo-secret-1
        property: password

EOF

kubectl apply -f externalsecret.yaml

###############################################################################

# 等待同步並列出結果
kubectl wait --for=condition=Ready externalsecret/demo-es-1 --timeout=30s -n $NAMESPACE
echo -n "demo-1-username: "
kubectl get secret sync-secret-1 -n $NAMESPACE -o jsonpath="{.data.demo-1-username}" | base64 --decode && echo
echo -n "demo-1-password: "
kubectl get secret sync-secret-1 -n $NAMESPACE -o jsonpath="{.data.demo-1-password}" | base64 --decode && echo

```

- Apply 後即會在對應 namespace 產生 secret
    
    ```yaml
    apiVersion: v1
    data:
      demo-1-password: ZGVtby1iYXNpYy11c2FnZQ==
      demo-1-username: ZGVtby11c2VyLTE=
    kind: Secret
    metadata:
      annotations:
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"external-secrets.io/v1beta1","kind":"ExternalSecret","metadata":{"annotations":{},"name":"demo-es-1","namespace":"default"},"spec":{"data":[{"remoteRef":{"key":"demo-secret-1","property":"username"},"secretKey":"demo-1-username"},{"remoteRef":{"key":"demo-secret-1","property":"password"},"secretKey":"demo-1-password"}],"refreshInterval":"15s","secretStoreRef":{"kind":"SecretStore","name":"demo-ss-1"},"target":{"name":"sync-secret-1"}}}
        reconcile.external-secrets.io/data-hash: 0dd137bbdbeab8634f02d36c2d0f8901
      creationTimestamp: "2025-04-16T13:11:51Z"
      labels:
        reconcile.external-secrets.io/created-by: 4a0ae8f7efadd80cfc0595c21ee3c879
        reconcile.external-secrets.io/managed: "true"
      name: sync-secret-1
      namespace: default
      ownerReferences:
      - apiVersion: external-secrets.io/v1
        blockOwnerDeletion: true
        controller: true
        kind: ExternalSecret
        name: demo-es-1
        uid: 640c390e-cb03-4392-98c2-8f68c45f7a63
      resourceVersion: "12246816"
      uid: 6cf450b5-5e5b-4470-93b1-087b04e187f8
    type: Opaque
    
    ```
    
- 可以觀察 logs 中會記錄 reconcile
    
    ```yaml
    ...
    ...
    ...
     {"level":"info","ts":1744809111.4586189,"logger":"controllers.ExternalSecret","msg":"reconciled secret","ExternalSecret":{"name":"demo-es-1","namespace":"default"}}
    ```
    
- 於 OpenBao 進行 key vaule 修改，看同步回來的值有沒有發生變化

## Clean up script

```bash
vault kv metadata delete secret/demo-secret-1

kubectl delete ns $NAMESPACE
```

# Useful Commands

```bash
SECRET_NAME=demo-secret-1
SECRET_VERSION=1

# 列出所有 kv secret
vault kv list secret/

# 取得 secret value
vault kv get -version=$SECRET_VERSION secret/$SECRET_NAME

# 刪除 (不含 metadata) (可復原)
vault kv delete secret/$SECRET_NAME
vault kv undelete -versions=$SECRET_VERSION secret/$SECRET_NAME

# 查看 secret 有哪些版本
vault kv metadata get secret/$SECRET_NAME

# 刪除 (不含 metadata) (不可復原)
# 刪除後仍可 get 到，但就不會包括 Data Section
vault kv destroy -versions=$SECRET_VERSION secret/$SECRET_NAME

# 刪除 (含 metadata) (不可復原)
vault kv metadata delete secret/$SECRET_NAME

```