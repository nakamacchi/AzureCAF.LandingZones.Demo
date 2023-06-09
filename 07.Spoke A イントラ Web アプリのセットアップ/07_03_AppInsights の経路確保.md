# AppInsights の経路確保

```bash
 
# NW 管理チームに依頼して AppInsights の経路を確保
# 解放する FQDN は以下
# https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/ip-addresses
 
# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# Hub サブスクリプションの Firewall に穴をあける
az account set -s "${SUBSCRIPTION_ID_HUB}"
 
TEMP_MAIN_LOCATION=${LOCATION_NAMES[0]}
TEMP_MAIN_RG_NAME="rg-hub-${LOCATION_PREFIXS[0]}"
TEMP_MAIN_FWP_NAME="fwp-fw-hub-${LOCATION_PREFIXS[0]}"
 
# Azure Firewall に Application Insights の追加の穴あけを行う
# https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/ip-addresses
 
# 共通ポリシーに対して設定
az network firewall policy rule-collection-group collection add-filter-collection \
	--resource-group ${TEMP_MAIN_RG_NAME} --policy-name "${TEMP_MAIN_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
	--name "Application Insights" --rule-type ApplicationRule --collection-priority 20500 --action Allow \
	--rule-name "Telemetories" \
	--target-fqdns "dc.applicationinsights.azure.com" "dc.applicationinsights.microsoft.com" "dc.services.visualstudio.com" "*.in.applicationinsights.azure.com" \
	--source-addresses "*" --protocols Https=443
 
az network firewall policy rule-collection-group collection rule add \
	--resource-group "${TEMP_MAIN_RG_NAME}" --policy-name "${TEMP_MAIN_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
	--collection-name "Application Insights" --rule-type ApplicationRule \
	--name "Live Metrics" \
	--target-fqdns "live.applicationinsights.azure.com" "rt.applicationinsights.microsoft.com" "rt.services.visualstudio.com" "*.livediagnostics.monitor.azure.com" \
	--source-addresses "*" --protocols Https=443
 
az network firewall policy rule-collection-group collection rule add \
	--resource-group "${TEMP_MAIN_RG_NAME}" --policy-name "${TEMP_MAIN_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
	--collection-name "Application Insights" --rule-type ApplicationRule \
	--name "Profiler" \
	--target-fqdns "profiler.monitor.azure.com" \
	--source-addresses "*" --protocols Https=443
 
```
