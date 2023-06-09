# Web App の作成

```bash
 
# 業務システム B チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spokeb_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# Spoke B サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_B}"
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
TEMP_ASP_WIN_NAME="aspw-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
 
# App Service Plan (Windows) の作成
az appservice plan create --name "${TEMP_ASP_WIN_NAME}" --resource-group "$TEMP_RG_NAME" --location "${TEMP_LOCATION_NAME}" --sku P1V2 --number-of-workers 2
 
# Web App の作成
# Web App 名はグローバルに一意である必要があるため UNIQUE_SUFFIX を付与する
TEMP_WEBAPP_NAME="webapp-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_ASP_WIN_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Web/serverFarms/${TEMP_ASP_WIN_NAME}"
az webapp create --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --plan $TEMP_ASP_WIN_ID
 
# システム割当 MID を有効化しておく　※ 本来はこの ID でリソースアクセスするのが望ましい
az webapp identity assign --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME
 
# タイムゾーンの設定
az webapp config appsettings set --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --settings "WEBSITE_TIME_ZONE=Tokyo Standard Time"
# FTP の無効化 (Windows 版では Kudu で事足りるので塞ぐ)
az webapp config set --ftps-state Disabled --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME}
# ログの有効化 (App Service Log)
az webapp log config --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --application-logging filesystem --detailed-error-messages true --failed-request-tracing true --web-server-logging filesystem --level warning
 
done

```
