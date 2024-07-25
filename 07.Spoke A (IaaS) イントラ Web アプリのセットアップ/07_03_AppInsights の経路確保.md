# AppInsights の経路確保

Application Insights は、物理データストアとして Log Analytics Workspace を利用しますが、Application Insights 特有の監視機能を使うために、追加でいくつかの穴開けが必要になります。以下を実行して、これらの穴開けを実施します。

```bash

# NW 管理チームに依頼して AppInsights の経路を確保
# 解放する FQDN は以下
# https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/ip-addresses

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# Hub サブスクリプションの Firewall に穴をあける
az account set -s "${SUBSCRIPTION_ID_HUB}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
TEMP_FWP_NAME="fw-hub-${TEMP_LOCATION_PREFIX}-fwp"

# Azure Firewall に Application Insights の追加の穴あけを行う
# https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/ip-addresses

# 共通ポリシーに対して設定
az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group ${TEMP_RG_NAME} --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
  --name "Application Insights" --rule-type ApplicationRule --collection-priority 20500 --action Allow \
  --rule-name "Telemetories" \
  --target-fqdns "dc.applicationinsights.azure.com" "dc.applicationinsights.microsoft.com" "dc.services.visualstudio.com" "*.in.applicationinsights.azure.com" \
  --source-addresses "*" --protocols Https=443

az network firewall policy rule-collection-group collection rule add \
  --resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
  --collection-name "Application Insights" --rule-type ApplicationRule \
  --name "Live Metrics" \
  --target-fqdns "live.applicationinsights.azure.com" "rt.applicationinsights.microsoft.com" "rt.services.visualstudio.com" "*.livediagnostics.monitor.azure.com" \
  --source-addresses "*" --protocols Https=443

az network firewall policy rule-collection-group collection rule add \
  --resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
  --collection-name "Application Insights" --rule-type ApplicationRule \
  --name "Profiler" \
  --target-fqdns "profiler.monitor.azure.com" \
  --source-addresses "*" --protocols Https=443

done # TEMP_LOCATION

```
