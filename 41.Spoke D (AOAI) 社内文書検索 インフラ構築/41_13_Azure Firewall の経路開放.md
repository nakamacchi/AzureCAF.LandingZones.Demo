# Azure Firewall の経路開放

Azure OpenAI Service を利用する際に必要となるライブラリやデータを取得するため、Web アプリから以下の FQDN への経路を行います。

- openaipublic.blob.core.windows.net

```bash

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ハブサブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_HUB}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
TEMP_FWP_NAME="fw-hub-${TEMP_LOCATION_PREFIX}-fwp"

TEMP_SUBNET_ASB_ADDRESS="${IP_SPOKE_D_PREFIXS[$i]}.2.0/24"

az network firewall policy rule-collection-group collection add-filter-collection \
--resource-group ${TEMP_RG_NAME} --policy-name ${TEMP_FWP_NAME} --rcg-name "DefaultApplicationRuleCollectionGroup" \
--name "Spoke D Web App" --rule-type ApplicationRule --collection-priority 40400 --action Allow \
--rule-name "Azure OpenAI Library" \
--target-fqdns "openaipublic.blob.core.windows.net" \
--source-addresses ${TEMP_SUBNET_ASB_ADDRESS} --protocols Https=443

done # TEMP_LOCATION

```
