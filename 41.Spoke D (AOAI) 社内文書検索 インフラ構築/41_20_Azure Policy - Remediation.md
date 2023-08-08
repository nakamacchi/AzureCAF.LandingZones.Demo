# Azure Policy - Remediation

Azure Policy により Non Compliant となった項目のうち、是正が可能な項目を是正します。

## Cosmos DB のローカル認証を無効化

- disableLocalAuth を false に変更します。
- 他の多くのリソースでは patch メソッドで処理できますが、2023/08 時点ではうまく処理できないため、ARM テンプレートで処理します。[詳細](https://docs.microsoft.com/azure/cosmos-db/how-to-setup-rbac#disable-local-auth)

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
# 下記の patch メソッドが 2023/08 時点では正しく動作しない
#TEMP_COSMOSDB_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/rg-spoked-${TEMP_LOCATION_PREFIX}/providers/Microsoft.DocumentDB/databaseAccounts/${TEMP_COSMOSDB_NAME}"
#az rest --method patch --uri "${TEMP_COSMOSDB_ID}?api-version=2023-04-15" --body @- <<EOF
#{
#    "properties": {
#        "disableLocalAuth": true
#    }
#}
#EOF
# このため、以下の ARM テンプレートで処理する

cat > temp.json <<EOF
{
    "\$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.DocumentDB/databaseAccounts",
            "apiVersion": "2023-04-15",
            "name": "${TEMP_COSMOSDB_NAME}",
            "location": "${TEMP_LOCATION_NAME}",
            "kind": "GlobalDocumentDB",
            "identity": {
                "type": "None"
            },
            "properties": {
                "publicNetworkAccess": "Disabled",
                "networkAclBypass": "AzureServices",
                "consistencyPolicy": {
                    "defaultConsistencyLevel": "Session",
                    "maxIntervalInSeconds": 5,
                    "maxStalenessPrefix": 100
                },
                "enableFreeTier": false,
                "locations": [
                    {
                        "locationName": "${TEMP_LOCATION_NAME}",
                        "failoverPriority": 0,
                        "isZoneRedundant": false
                    }
                ],
                "databaseAccountOfferType": "Standard",
                "disableLocalAuth": true
            }
        }
    ]
}
EOF
az deployment group create --name ${TEMP_COSMOSDB_NAME} --resource-group "${TEMP_RG_NAME}" --template-file temp.json

done # TEMP_LOCATION

```
