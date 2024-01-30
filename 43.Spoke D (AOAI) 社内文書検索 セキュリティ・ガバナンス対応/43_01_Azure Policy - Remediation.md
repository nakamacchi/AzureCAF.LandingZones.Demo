# Azure Policy - Remediation

Azure Policy により Non Compliant となった項目のうち、是正が可能な項目を是正します。

- Cosmos DB のローカル認証を無効化
- AOAI のローカル認証を無効化
- Form Recognizer のローカル認証を無効化
  - disableLocalAuth を false に変更します。[詳細](https://docs.microsoft.com/azure/cosmos-db/how-to-setup-rbac#disable-local-auth)

```bash

# 業務システム D チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"
 
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_COSMOSDB_NAME="cos-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_COSMOSDB_SQLDB_NAME="ChatHistory"
TEMP_COSMOSDB_CONTAINER_NAME="Prompts"

# ローカルキー認証を無効化
TEMP_COSMOSDB_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/rg-spoked-${TEMP_LOCATION_PREFIX}/providers/Microsoft.DocumentDB/databaseAccounts/${TEMP_COSMOSDB_NAME}"
az rest --method patch --uri "${TEMP_COSMOSDB_ID}?api-version=2023-04-15" --body @- <<EOF
{
    "properties": {
        "disableLocalAuth": true
    }
}
EOF

TEMP_AOAI_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/rg-spoked-${TEMP_LOCATION_PREFIX}/providers/Microsoft.CognitiveServices/accounts/aoai-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
az rest --method patch --uri "${TEMP_AOAI_ID}?api-version=2023-05-01" --body @- <<EOF
{
    "properties": {
        "disableLocalAuth": true
    }
}
EOF

TEMP_FORMRECOGNIZER_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/rg-spoked-${TEMP_LOCATION_PREFIX}/providers/Microsoft.CognitiveServices/accounts/fmr-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
az rest --method patch --uri "${TEMP_FORMRECOGNIZER_ID}?api-version=2023-05-01" --body @- <<EOF
{
    "properties": {
        "disableLocalAuth": true
    }
}
EOF

done # TEMP_LOCATION

```
