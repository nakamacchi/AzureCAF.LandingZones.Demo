# AppInsights 作成と有効化

```bash
# AppInsights をあとから有効化する場合、App Service 基盤は NuGet から動的にパッケージを取得する
# NuGet 全体への経路開放が必要になるため、実際には開発時にアプリに AppInsights を仕込んでからデプロイすることを推奨
 
##############################
 
# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# ハブサブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_HUB}"
 
TEMP_MAIN_LOCATION=${LOCATION_NAMES[0]}
TEMP_MAIN_RG_NAME="rg-hub-${LOCATION_PREFIXS[0]}"
TEMP_MAIN_FWP_NAME="fwp-fw-hub-${LOCATION_PREFIXS[0]}"
 
TEMP_SOURCE_ADDRESSES=""
for i in ${VDC_NUMBERS}; do
  TEMP_IP_PREFIX=${IP_SPOKE_B_PREFIXS[$i]}
  TEMP_SUBNET_ASB="${TEMP_IP_PREFIX}.10.0/24"
  TEMP_SOURCE_ADDRESSES+="${TEMP_SUBNET_ASB} "
done
echo $TEMP_SOURCE_ADDRESSES
 
az network firewall policy rule-collection-group collection add-filter-collection \
	--resource-group ${TEMP_MAIN_RG_NAME} --policy-name ${TEMP_MAIN_FWP_NAME} --rcg-name "DefaultApplicationRuleCollectionGroup" \
	--name "AppService" --rule-type ApplicationRule --collection-priority 40100 --action Allow \
	--rule-name "NuGet" \
	--target-fqdns "www.nuget.org" \
	--source-addresses ${TEMP_SOURCE_ADDRESSES} --protocols Https=443
 
##############################
 
# 業務システム B チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spokeb_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# Spoke B サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_B}"
 
# Application Insighs の作成と自動インストルメンテーション機能の有効化
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
  TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
  TEMP_APP_NAME="app-spokeb-${TEMP_LOCATION_PREFIX}"
 
  TEMP_LAW_NAME="law-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_LAW_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/${TEMP_LAW_NAME}"
 
az monitor app-insights component create --app ${TEMP_APP_NAME} --location ${TEMP_LOCATION_NAME} --kind web --resource-group ${TEMP_RG_NAME} --workspace ${TEMP_LAW_RESOURCE_ID}
 
TEMP_WEBAPP_NAME="webapp-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_WEBAPP_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Web/sites/${TEMP_WEBAPP_NAME}"
 
az monitor app-insights component connect-webapp --app ${TEMP_APP_NAME} --resource-group ${TEMP_RG_NAME} --web-app ${TEMP_WEBAPP_ID} --enable-debugger false --enable-profiler false
 
done
 
# Azure Portal の Web App 管理画面の Application Insights から Enable Application Insights を行い、各種のプロパティを設定する
 
```
