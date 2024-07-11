# AppInsights 作成と有効化

Web App 上に展開したアプリの詳細監視のため、Application Insights の機能を有効化します。

- 業務システム A の場合と同様に、Application Insights を利用することで、アプリケーションの内部まで踏み込んだ詳細解析が可能になります。
- なお Web App の自動インストルメンテーション機能を利用する場合には、以下の点に注意してください。
  - 自動インストルメンテーション機能を利用することにより、すでに配置済みのアプリケーションの詳細な内部稼働データの取得が可能になりますが、この処理を行うために、Web App ランタイムは NuGet から Application Insights のパッケージモジュールを動的に取得します。この際、当該 NuGet パッケージに限定した経路開放ができない（＝nuget.org 全体への経路開放が必要になる）ため、厳格な閉域化制御を行いたい場合には問題になります。
  - このような場合には、アプリケーションをビルドする際に Application Insights のモジュールをアプリ内に取り込んでしまってください。このようにすれば、デプロイ後の自動インストルメンテーション機能の利用は不要になります。
  - （本デモでは、すでにビルド済みのアプリを利用するため、nuget.org への経路を開放しています。）

```bash

# AppInsights をあとから有効化する場合、App Service 基盤は NuGet から動的にパッケージを取得する
# NuGet 全体への経路開放が必要になるため、実際には開発時にアプリに AppInsights を仕込んでからデプロイすることを推奨

##############################

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ハブサブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_HUB}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
TEMP_FWP_NAME="fw-hub-${TEMP_LOCATION_PREFIX}-fwp"

TEMP_SUBNET_ASB_ADDRESS="${IP_SPOKE_B_PREFIXS[$i]}.10.0/24"

az network firewall policy rule-collection-group collection add-filter-collection \
--resource-group ${TEMP_RG_NAME} --policy-name ${TEMP_FWP_NAME} --rcg-name "DefaultApplicationRuleCollectionGroup" \
--name "AppService" --rule-type ApplicationRule --collection-priority 40100 --action Allow \
--rule-name "NuGet" \
--target-fqdns "www.nuget.org" \
--source-addresses ${TEMP_SUBNET_ASB_ADDRESS} --protocols Https=443

done # TEMP_LOCATION

##############################

# 業務システム B チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokeb_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokeb_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

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

done # TEMP_LOCATION

# Azure Portal の Web App 管理画面の Application Insights から Enable Application Insights を行い、各種のプロパティを設定する

```
