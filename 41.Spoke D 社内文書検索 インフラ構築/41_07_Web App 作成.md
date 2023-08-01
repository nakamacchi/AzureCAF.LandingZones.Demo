# Web App 作成

Web App を作成します。

```bash

# 業務システム D チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_ASP_LINUX_NAME="aspl-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"

# App Service Plan (Linux) の作成
# ※ Windows 版は Python が 3.4 までしかサポートされていないため、今回のケースでは利用できない
# ※ Python 3.10 を利用するために Linux 版の App Service Plan を利用する。
az appservice plan create --name "${TEMP_ASP_LINUX_NAME}" --resource-group "$TEMP_RG_NAME" --location "${TEMP_LOCATION_NAME}" --sku P1V2 --number-of-workers 2 --is-linux

# Web App の作成
# Web App 名はグローバルに一意である必要があるため UNIQUE_SUFFIX を付与する
TEMP_WEBAPP_NAME="webapp-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_ASP_LINUX_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Web/serverFarms/${TEMP_ASP_LINUX_NAME}"
az webapp create --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --plan $TEMP_ASP_LINUX_ID --runtime "python|3.10"

# システム割当 MID を有効化しておく
az webapp identity assign --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME

# ログの有効化 (App Service Log)
az webapp log config --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --application-logging filesystem --detailed-error-messages true --failed-request-tracing true --web-server-logging filesystem --level warning

done # TEMP_LOCATION

```
