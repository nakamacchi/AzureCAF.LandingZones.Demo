# Spoke D リソース削除

Spoke D のリソースだけを削除したい場合には以下を実行します。

```bash

for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_ID_SPOKE_D; do
echo "Deleting environment : ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"
 
# リソースロック開放
for TEMP_RG_NAME in $(az group list --query "[].name" --output tsv); do
echo "Searching resource locks in $TEMP_RG_NAME"
 
# for TEMP_RES_ID in $(az resource list --resource-group $TEMP_RG_NAME --query "[].id" --output tsv); do
TEMP_IDS=$(az resource list --resource-group $TEMP_RG_NAME --query "[].id")
TEMP_IDS=${TEMP_IDS//  \"/\"}
TEMP_IDS=${TEMP_IDS//[\[\],]/}
TEMP_IDS=${TEMP_IDS// /SPACEFIX}
for TEMP in $TEMP_IDS; do
TEMP_ID="${TEMP//SPACEFIX/ }"
TEMP_RES_ID="${TEMP_ID//\"/}"
 
for TEMP_RES_LOCK_ID in $(az resource lock list --resource $TEMP_RES_ID --query "[].id" --output tsv); do
echo "Deleting ${TEMP_RES_LOCK_ID}"
az resource lock delete --ids $TEMP_RES_LOCK_ID
done #TEMP_RES_LOCK_ID
 
done #TEMP_RES_ID
done #TEMP_RG_NAME
 
# リソース削除を開始 (非同期)
for TEMP_RG_NAME in $(az group list --query "[].name" --output tsv); do
echo $TEMP_RG_NAME
az group delete --yes --resource-group $TEMP_RG_NAME --no-wait
done #TEMP_RG_NAME
done #TEMP_SUBSCRIPTION
 
# リソース削除を実施（同期）
for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_ID_SPOKE_D; do
echo "Waiting deletion environment : ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"
for TEMP_RG_NAME in $(az group list --query "[].name" --output tsv); do
echo $TEMP_RG_NAME
az group delete --yes --resource-group $TEMP_RG_NAME
done #TEMP_RG_NAME
done #TEMP_SUBSCRIPTION

# ソフトデリートされたリソースをチェックしてパージ
# パージ処理の書き方が少し異なる
for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_ID_SPOKE_D; do
echo "Purging deleted resources : ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"

# Cognitive Services
# https://learn.microsoft.com/en-us/rest/api/cognitiveservices/accountmanagement/deleted-accounts/purge?tabs=HTTP
TEMP_RES_IDS=$(az cognitiveservices account list-deleted --query "[].id" -o tsv)
if [[ -n $TEMP_RES_IDS ]]; then
  for TEMP_RES_ID in ${TEMP_RES_IDS}; do
    echo $TEMP_RES_ID
    az rest --method DELETE --url "${TEMP_RES_ID}?api-version=2021-10-01"
  done
fi
# Key Vault
# https://learn.microsoft.com/en-us/rest/api/keyvault/keyvault/vaults/purge-deleted?tabs=HTTP
TEMP_RES_IDS=$(az keyvault list-deleted --query "[].id" -o tsv)
echo $TEMP_RES_IDS
if [[ -n $TEMP_RES_IDS ]]; then
  for TEMP_RES_ID in ${TEMP_RES_IDS}; do
    echo $TEMP_RES_ID
    az rest --method POST --url "${TEMP_RES_ID}/purge?api-version=2022-07-01"
  done
fi

done #TEMP_SUBSCRIPTION

```
