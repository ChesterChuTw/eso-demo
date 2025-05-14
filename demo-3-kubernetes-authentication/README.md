# Core Concept

- 將 Pod 的「Service Account 身份」轉換為 Vault 可接受的認證方式
- 也就是用 **Kubernetes Auth Method** 讓 Vault 信任該 Pod 身份，並授權它存取特定的 Secrets

# 架構流程簡述

1. **在 Vault 註冊一個 role**，綁定對應的 Service Account
2. **Pod 使用該 Service Account 執行**
3. **ESO 自動用該身份向 Vault 登入並拉取 secrets**
4. **Vault Secrets Map 到 Kubernetes Secret，提供給應用程式**

# OpenBao

## **Prerequisite**

```bash
# 啟用必要服務
bao secrets enable -path=secret kv-v2
bao auth enable kubernetes
```

## Flow

```bash
#!/bin/bash

NAMESPACE="demo3"
VAULT_SERVER_ENDPOINT="http://192.168.1.240:8200"
K8S_SERVER_ENDPOINT="https://192.168.1.200:6443"

set -e

# Vault Demo Secret & Policy Creation

## 建立 demo db user/password
bao kv put secret/demo3/database password="s3cr3tP@ssw0rd" username="dbuser"

## 建立可以存取 demo3 的 policy 
cat <<EOF > demo3-policy.hcl
path "secret/data/demo3/*" {
  capabilities = ["read"]
}
EOF

# 建立 policy
vault policy write demo3-policy demo3-policy.hcl

#################################################################################

# Vault -> Kubernetes 驗證設定

kubectl create ns $NAMESPACE

## Service Account for Vault token review
kubectl create serviceaccount vault-auth -n $NAMESPACE

## 建立 RoleBinding，讓 Service Account vault-auth 可以 call TokenReview
cat <<EOF > token-reviewer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-auth-tokenreview-binding
subjects:
- kind: ServiceAccount
  name: vault-auth
  namespace: $NAMESPACE
roleRef:
  kind: ClusterRole
  name: system:auth-delegator
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f token-reviewer.yaml

## 建立 Vault 所需的 Service Account Token Secret
cat <<EOF > vault-auth-token.yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-token
  namespace: $NAMESPACE
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
EOF

kubectl apply -f vault-auth-token.yaml

sleep 5

## 取得 JWT 與 CA 證書
kubectl get secret vault-auth-token -n $NAMESPACE -o jsonpath="{.data.token}" | base64 -d > vault-auth-token.jwt
kubectl get secret vault-auth-token -n $NAMESPACE -o jsonpath="{.data['ca\.crt']}" | base64 -d > ca.crt

## 設定 Vault 與 Kubernetes 的認證溝通
vault write auth/kubernetes/config \
  token_reviewer_jwt=@vault-auth-token.jwt \
  kubernetes_host="$K8S_SERVER_ENDPOINT" \
  kubernetes_ca_cert=@ca.crt

#################################################################################

# Demo App 設定

## Service Account for Pod/Deployment Access
kubectl create serviceaccount demo3-sa -n $NAMESPACE

## 綁定 Pod/Deployment 使用這個 Service Account
### 該 Pod 啟動後會自動掛載一組 JWT Token 與 CA 憑證 到 `/var/run/secrets/kubernetes.io/serviceaccount/`
cat <<EOF > nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo3-app
  namespace: $NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo3-app
  template:
    metadata:
      labels:
        app: demo3-app
    spec:
      serviceAccountName: demo3-sa  # <<< 
      containers:
      - name: app
        image: nginx
EOF

kubectl apply -f nginx.yaml

## 在 Vault 建立 Vault Role 並綁定 Policy & Kubernetes Service Account
vault write auth/kubernetes/role/demo3 \
  bound_service_account_names="demo3-sa" \
  bound_service_account_namespaces="$NAMESPACE" \
  policies="demo3-policy" \
  ttl="24h"

#################################################################################

# Kubernetes ESO 設定

## SecretStore (指定驗證方式、SA...)
cat <<EOF > vault-secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: $NAMESPACE
spec:
  provider:
    vault:
      server: "$VAULT_SERVER_ENDPOINT"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "demo3"
          serviceAccountRef:
            name: demo3-sa
EOF

kubectl apply -f vault-secret-store.yaml

# ExternalSecret (指定要同步的資料)
cat <<EOF > demo3-external-secrets.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: demo3-external-secrets
  namespace: $NAMESPACE
spec:
  refreshInterval: 15s
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: demo3-sync-secret
  data:
    - secretKey: db-password
      remoteRef:
        key: demo3/database
        property: password
EOF

kubectl apply -f demo3-external-secrets.yaml

kubectl -n $NAMESPACE wait --for=condition=Ready externalsecret/demo3-external-secrets --timeout=30s
DB_PASS=$(kubectl -n $NAMESPACE get secret demo3-sync-secret -o jsonpath="{.data.db-password}" | base64 -d)

echo "DB_PASSWORD: $DB_PASS"

```

## Clean up script

```bash
#!/bin/bash

set -e

NAMESPACE="demo3"

echo "🧹 開始清除 Kubernetes 資源..."

kubectl delete clusterrolebinding vault-auth-tokenreview-binding --ignore-not-found
kubectl delete namespace $NAMESPACE --ignore-not-found

echo "✅ Kubernetes namespace $NAMESPACE 已刪除（包含 SA、Pods、Secrets 等）"

echo "🧹 開始清除 Vault 設定..."

# 刪除 Vault policy
vault policy delete demo3-policy || true

# 刪除 Vault role
vault delete auth/kubernetes/role/demo3 || true

# 刪除 Vault KV Secret
vault kv metadata delete secret/demo3/database || true

echo "✅ Vault 設定已清除"

echo "🗑 清除本地暫存檔案..."
# rm -f \
#   demo3-policy.hcl \
#   token-reviewer.yaml \
#   vault-auth-token.yaml \
#   vault-auth-token.jwt \
#   ca.crt \
#   nginx.yaml \
#   vault-secret-store.yaml \
#   demo3-external-secrets.yaml

echo "✅ 所有清除作業完成"

```

## Vault 向 Kubernetes 執行 TokenReview 認證

- Vault 在向 Kubernetes 驗證 Pod 的 ServiceAccount 時，需要具備 相應的權限（透過 `token_reviewer_jwt`）以及 Kubernetes 的 CA 憑證（`kubernetes_ca_cert` 用於 TLS 驗證）才能成功執行 TokenReview 認證
- token_reviewer_jwt
    - Kubernetes ServiceAccount 的 JWT Token
    - 用來讓 Vault 向 Kubernetes API Server 做 TokenReview 認證時，能擁有合法身分與權限
    - 為什麼要有 `token_reviewer_jwt`？
        - Vault 拿到 Pod 傳來的 ServiceAccount Token（JWT）後，**不能自己驗證它是否有效**
            - 所以 Vault 會呼叫 [Kubernetes API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-review-v1/) `/apis/authentication.k8s.io/v1/tokenreviews`
        - `tokenreviews` API **需要身分與權限**才能執行
            - 所以 Vault 自己需要一個有權限的 ServiceAccount Token（這就是 `token_reviewer_jwt`）
- kubernetes_ca_cert
    - Vault 要透過 HTTPS 存取 Kubernetes API Server
    - 若不提供 CA 憑證，Vault 無法確認連線目標是不是合法的 Kubernetes Server（會被 TLS 拒絕）
    - 所以你必須提供 Kubernetes 的 CA 憑證（通常從 ServiceAccount Secret 中提取）

# Useful Commands

```bash
# SECRET-Related Commands

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

# 刪除 (含 metadata) (不可復原) (不會遞迴刪除)
vault kv metadata delete secret/$SECRET_NAME

```

```bash
# APPROLE-Related Commands

ROLE_NAME=ci-deployer

# 列出所有 approle 名稱
vault list auth/approle/role

# 取得 role 資訊
vault read auth/approle/role/$ROLE_NAME

# 更改 policy
vault write auth/approle/role/$ROLE_NAME \
  token_policies="default,ci-policy,login-policy"
  
# 刪除 approle
vault delete auth/approle/role/$ROLE_NAME  
  
# 查看當前角色資訊
bao token lookup

```

```bash
# POLICY-Related Commands

POLICY_NAME=ci-policy

# 列出所有 policy
vault policy list

# 查看 policy
vault policy read $POLICY_NAME

# 刪除 policy
vault policy delete $POLICY_NAME
```