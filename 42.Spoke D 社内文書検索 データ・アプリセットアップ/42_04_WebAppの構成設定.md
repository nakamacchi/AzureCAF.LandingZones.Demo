
```bash

# 業務システム D チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
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

#構成設定

TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_APP_NAME="app-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_APPINSIGHTS_INSTRUMENATION_KEY=$(az monitor app-insights component show --app ${TEMP_APP_NAME} --resource-group ${TEMP_RG_NAME} --query 'instrumentationKey' -o tsv)

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
"APPLICATIONINSIGHTS_CONNECTION_STRING=${TEMP_APPINSIGHTS_INSTRUMENATION_KEY}"

az webapp config container set --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --docker-custom-image-name $TEMP_IMAGE_NAME --docker-registry-server-url "https://${TEMP_ACR_NAME}.azurecr.io" --vnet-route-all-enabled 1

#下記のコマンドだとうまくいかない
#az webapp config container set --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --docker-custom-image-name ${TEMP_IMAGE_NAME} --docker-registry-server-url ${TEMP_ACR_NAME}.azurecr.io
#GUI から設定すると下記
#{"requests":[{"httpMethod":"PATCH","content":{"identity":{"type":"SystemAssigned"}},"requestHeaderDetails":{"commandName":"enableSystemAssignedIdentity"},"url":"/subscriptions/5399cafa-d5c8-4ffd-821c-7e7673e6d37e/resourceGroups/rg-spoked-eus/providers/Microsoft.Web/sites/webapp-spoked-07541-eus?api-version=2021-02-01"},{"httpMethod":"PUT","content":{"id":"/subscriptions/5399cafa-d5c8-4ffd-821c-7e7673e6d37e/resourceGroups/rg-spoked-eus/providers/Microsoft.Web/sites/webapp-spoked-07541-eus/config/appsettings","name":"appsettings","type":"Microsoft.Web/sites/config","location":"East US","properties":{"WEBSITE_HTTPLOGGING_RETENTION_DAYS":"3","WEBSITE_VNET_ROUTE_ALL":"1","WEBSITE_DNS_SERVER":"10.0.251.4","WEBSITES_ENABLE_APP_SERVICE_STORAGE":"false","DOCKER_REGISTRY_SERVER_USERNAME":"acrspoked07541eus","DOCKER_REGISTRY_SERVER_PASSWORD":"LDQB5H04/ng+KSvbjUIEio2qaynifoCz2I8S49Ujxx+ACRBwUoY1","DOCKER_REGISTRY_SERVER_URL":"https://acrspoked07541eus.azurecr.io"}},"requestHeaderDetails":{"commandName":"updateApplicationSettings"},"url":"/subscriptions/5399cafa-d5c8-4ffd-821c-7e7673e6d37e/resourceGroups/rg-spoked-eus/providers/Microsoft.Web/sites/webapp-spoked-07541-eus/config/appsettings?api-version=2018-11-01"}]}

#ACR 取得は VNET 統合が効かないので outbound IP アドレスを有効化する必要がある？
#20.119.16.39
#20.124.77.214


done # TEMP_LOCATION



```
