# Web App の構成設定

Web App に対して以下を設定します。

- アプリケーションを動作させるために必要な環境変数を Web App の構成設定として作成
- Web App の動作を完全閉域化（ACR Pull も閉域動作するように設定）
- Web App に対して取得するコンテナの設定

```bash

# 業務システム D チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spoked_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
 
# Spoke D サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_CA_NAME="ca-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_ACR_NAME="acrspoked${UNIQUE_SUFFIX}${TEMP_LOCATION_PREFIX}"
TEMP_IMAGE_NAME="${TEMP_ACR_NAME}.azurecr.io/aoai.sample4:latest"

TEMP_STORAGE_NAME="stspoked${UNIQUE_SUFFIX}${TEMP_LOCATION_PREFIX}"
TEMP_COSMOSDB_NAME="cos-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_SEARCHSERVICE_NAME="src-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_FORMRECOGNIZER_NAME="fmr-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_AOAI_NAME="aoai-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"

TEMP_COSMOSDB_SQLDB_NAME="ChatHistory"
TEMP_COSMOSDB_CONTAINER_NAME="Prompts"
TEMP_AOAI_DAVINCI_DEPLOYMENT_NAME="davinci"
TEMP_AOAI_GPT_35_TURBO_DEPLOYMENT_NAME="chat"
TEMP_AOAI_GPT_4_32K_DEPLOYMENT=''
TEMP_AOAI_GPT_4_DEPLOYMENT=''
TEMP_AOAI_API_VERSION='2023-05-15'

TEMP_IMAGE_NAME="${TEMP_ACR_NAME}.azurecr.io/aoai.sample4:latest"

TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_APP_NAME="app-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_APPINSIGHTS_CONNECTION_STRING=$(az monitor app-insights component show --app ${TEMP_APP_NAME} --resource-group ${TEMP_RG_NAME} --query 'connectionString' -o tsv)

# 環境変数の設定
az webapp config appsettings set --name ${TEMP_WEBAPP_NAME} --resource-group $TEMP_RG_NAME --settings \
"AZURE_STORAGE_ACCOUNT=${TEMP_STORAGE_NAME}" \
"AZURE_STORAGE_CONTAINER=content" \
"AZURE_SEARCH_SERVICE=${TEMP_SEARCHSERVICE_NAME}" \
"AZURE_SEARCH_INDEX=gptkbindex" \
"AZURE_OPENAI_SERVICE=${TEMP_AOAI_NAME}" \
"AZURE_OPENAI_API_VERSION=2023-05-15" \
"AZURE_OPENAI_DAVINCI_DEPLOYMENT=${TEMP_AOAI_DAVINCI_DEPLOYMENT_NAME}" \
"AZURE_OPENAI_GPT_35_TURBO_DEPLOYMENT=${TEMP_AOAI_GPT_35_TURBO_DEPLOYMENT_NAME}" \
"AZURE_OPENAI_GPT_4_DEPLOYMENT=${TEMP_AOAI_GPT_4_DEPLOYMENT}" \
"AZURE_OPENAI_GPT_4_32K_DEPLOYMENT=${TEMP_AOAI_GPT_4_32K_DEPLOYMENT}" \
"AZURE_COSMOSDB_CONTAINER=${TEMP_COSMOSDB_CONTAINER_NAME}" \
"AZURE_COSMOSDB_DATABASE=${TEMP_COSMOSDB_SQLDB_NAME}" \
"AZURE_COSMOSDB_ENDPOINT=https://${TEMP_COSMOSDB_NAME}.documents.azure.com" \
"APPLICATIONINSIGHTS_CONNECTION_STRING=${TEMP_APPINSIGHTS_CONNECTION_STRING}"

# 閉域動作の設定
# https://learn.microsoft.com/ja-jp/azure/app-service/configure-custom-container?tabs=debian&pivots=container-linux
# https://learn.microsoft.com/ja-jp/azure/app-service/configure-vnet-integration-routing
# WEBSITE_PULL_IMAGE_OVER_VNET
az webapp config set --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --vnet-route-all-enabled

# ACR からのコンテナ取得に VNET と MID を利用する
az webapp config set --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --generic-configurations '{"acrUseManagedIdentityCreds": true}'
az resource update --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --resource-type "Microsoft.Web/sites" --set properties.vnetImagePullEnabled=true

# 取得するコンテナを指定
az webapp config container set --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --docker-custom-image-name $TEMP_IMAGE_NAME --docker-registry-server-url "https://${TEMP_ACR_NAME}.azurecr.io"

done # TEMP_LOCATION

```
