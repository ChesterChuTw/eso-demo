[使用 AppRole 實現 Vault 動態密碼注入至 Kubernetes (CI/CD)]

## 🔐 核心概念

- AppRole 是為了「**短期、動態、非人類、自動化流程**」所設計的認證方式
    - E.g., CI/CD Pipeline

# 🎯 應用場景流程（CI/CD Pipeline）

1. CI Job 開始時透過 Vault CLI/API 產出新的 secret_id（一次性）
2. 把 secret_id 寫入 K8s Secret（供 ESO 使用）
3. 建立 Temporary ExternalSecret + SecretStore
4. 由 ESO 拉取 secret 寫入 Kubernetes Secret
5. 應用程式/Job 使用該 Secret
6. 同步完成後清除資源（Secret、SecretStore、ExternalSecret）

# OpenBao

### **Prerequisite**

```bash
bao secrets enable -path=secret kv-v2
bao auth enable approle 
```

### Flow: dynamic deploy token obtainment in CI/CD pipeline

```bash
NAMESPACE=demo2
ROLE_NAME=ci-deployer

# 建立 Secret & Policy

## create a demo deploy token
bao kv put secret/ci/deploy-token password=tmp1234567

## 可讀取該 Secret 的 Policy
cat <<EOF > ci-policy.hcl
path "secret/data/ci/deploy-token" {
  capabilities = ["read"]
}
EOF

## 登入 approle 所需權限
cat <<EOF > login-policy.hcl
path "auth/approle/login" {
  capabilities = ["create", "read", "update"]
}
EOF

vault policy write ci-policy ci-policy.hcl
vault policy write login-policy login-policy.hcl

#############################################################################

# 建立 Vault AppRole

## create role and bind policy
###  token_ttl="20m" \                    # ← 🔹取得的 token 活 20 分鐘
###  token_type="batch" \                # ← 🔹代表發出的 token 無法續期，輕量短命
###  secret_id_ttl="10m" \               # ← 🔹secret_id 可使用 10 分鐘內
###  secret_id_num_uses=3                # ← 🔹最多使用 3 次（超過就作廢）
vault write auth/approle/role/$ROLE_NAME \
  token_policies="default,login-policy,ci-policy" \
  token_ttl="20m" \
  token_type="batch" \
  secret_id_ttl="10m" \
  secret_id_num_uses=3


## 取得 ROLE_ID & SECRET_ID
ROLE_ID=$(vault read -format=json "auth/approle/role/$ROLE_NAME/role-id" | jq -r '.data.role_id')
SECRET_ID=$(vault write -f -format=json auth/approle/role/$ROLE_NAME/secret-id | jq -r '.data.secret_id')

echo "ROLE_ID: $ROLE_ID"
echo "SECRET_ID: $SECRET_ID"

# Kubernetes ESO Resources

kubectl create ns $NAMESPACE

## 建立 secret 存放 secret_id 
cat <<EOF > vault-secret-id.yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-secret-id
  namespace: ${NAMESPACE}
type: Opaque
stringData:
  secretId: "${SECRET_ID}"
EOF

kubectl apply -f vault-secret-id.yaml

 ## SecretStore (指定驗證方式) 
cat <<EOF > secretstore.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-ci-store
  namespace: ${NAMESPACE}
spec:
  provider:
    vault:
      server: "http://192.168.1.240:8200"
      path: "secret"
      version: "v2"
      auth:
        appRole:
          path: approle
          roleId: "${ROLE_ID}"
          secretRef:
            name: vault-secret-id
            key: secretId
EOF

kubectl apply -f secretstore.yaml

# ExternalSecret (指定要同步的資料)

cat <<EOF > externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-ci-secret
  namespace: ${NAMESPACE}
spec:
  refreshInterval: "10m"
  secretStoreRef:
    name: vault-ci-store
    kind: SecretStore
  target:
    name: ci-synced-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: ci/deploy-token
        property: password
EOF

kubectl apply -f externalsecret.yaml

# 等待同步完成 & 讀取 secret
kubectl -n $NAMESPACE wait --for=condition=Ready externalsecret/vault-ci-secret --timeout=30s
DEPLOY_TOKEN=$(kubectl -n $NAMESPACE get secret ci-synced-secret -o jsonpath="{.data.password}" | base64 -d)

echo "DEPLOY_TOKEN:$DEPLOY_TOKEN"

```

- `vault write auth/approle/role` 參數說明
    
    `token_policies` 指定產生的 token 所擁有的權限
    
    ### `bind_secret_id`（預設：`true`）
    
    - **功能**：指定是否在登入時需要提供 `secret_id`。
    - **建議**：除非有其他安全限制（如 IP 限制），否則應保持為 `true`，以增強安全性。
    
    ### `secret_id_ttl`
    
    - **功能**：設定 `secret_id` 的有效期限。
    - **格式**：可使用秒數（如 `3600`）或時間單位（如 `60m`）。
    - **建議**：根據應用需求設定，短期使用建議設為較短的 TTL。
    
    ### `secret_id_num_uses`
    
    - **功能**：設定每個 `secret_id` 可使用的次數。
    - **預設值**：`0`（無限次數）。
    - **建議**：為防止濫用，建議設定為有限次數，如 `1` 或 `10`。
    
    ### `secret_id_bound_cidrs`
    
    - **功能**：限制可使用 `secret_id` 的 IP 範圍。
    - **格式**：CIDR 格式的 IP 範圍列表。
    - **建議**：在可能的情況下，設定為應用所在的 IP 範圍，以增強安全性。
    
    ### `token_type`
    
    - **選項**：
        - `default`：可續期的服務型 token。
        - `batch`：不可續期的輕量型 token，適用於短期使用。
    - **建議**：自動化流程中，建議使用 `batch`，以減少資源消耗。
    
    ### `token_ttl`
    
    - **功能**：設定發行的 token 的初始有效期限。
    - **格式**：可使用秒數或時間單位。
    - **建議**：根據應用需求設定，短期使用建議設為較短的 TTL。
    
    ### `token_max_ttl`
    
    - **功能**：設定 token 的最大有效期限，即使續期也不會超過此值。
    - **建議**：設定為合理的最大值，以防止 token 長期有效。
    
    ### `token_num_uses`
    
    - **功能**：設定 token 可使用的次數。
    - **預設值**：`0`（無限次數）。
    - **建議**：若希望限制 token 的使用次數，可設定為有限次數。
    
    ### `token_bound_cidrs`
    
    - **功能**：限制 token 可使用的 IP 範圍。
    - **格式**：CIDR 格式的 IP 範圍列表。
    - **建議**：在可能的情況下，設定為應用所在的 IP 範圍，以增強安全性。
    
    ### `token_no_default_policy`
    
    - **功能**：是否在發行的 token 中排除 `default` policy。
    - **預設值**：`false`（包含 `default` policy）。
    - **建議**：若希望精確控制 token 的權限，可設為 `true`，並明確指定所需的 policies。

# Clean up script

```bash

NAMESPACE=demo2
ROLE_NAME=ci-deployer

kubectl delete ns $NAMESPACE

vault policy delete ci-policy || true
vault policy delete login-policy || true
vault delete auth/approle/role/$ROLE_NAME
vault kv metadata delete secret/ci/deploy-token || true
```

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

```bash
ROLE_ID=$(vault read -format=json "auth/approle/role/ci-deployer/role-id" | jq -r '.data.role_id')
SECRET_ID=$(vault write -f -format=json auth/approle/role/ci-deployer/secret-id | jq -r '.data.secret_id')

bao write auth/approle/login \
  role_id="$ROLE_ID" \
  secret_id="$SECRET_ID"
  
  
VAULT_TOKEN="s.qB8nAzNXgo2gZSUA2DsRCkot"
```