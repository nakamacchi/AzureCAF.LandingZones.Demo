# Managed Prometheus のための穴あけ

```bash

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_AMW_NAME="maw-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_GRF_NAME="grf-vdc-${TEMP_LOCATION_PREFIX}"

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# FQDN 要件
# https://learn.microsoft.com/ja-jp/azure/azure-monitor/containers/kubernetes-monitoring-firewall

# 解放すべき FQDN を収集
# Azure Monitor Workspace に紐づいている DCE を取得して FQDN を取り出す
# 基本的には以下の 2 つ
# rg-spokee-jpe / MSProm-eastus-aks-spokee-jpe
# MA_maw-vdc-jpe_japaneast_managed / maw-vdc-jpe

az account set -s "${SUBSCRIPTION_ID_MGMT}"
TEMP_DCE_RES_IDS1=$(az resource list --resource-type "Microsoft.Insights/dataCollectionEndpoints" --query [].id -o tsv)

az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"
TEMP_DCE_RES_IDS2=$(az resource list --resource-type "Microsoft.Insights/dataCollectionEndpoints" --resource-group "rg-spokee-${TEMP_LOCATION_PREFIX}" --query [].id -o tsv)

TEMP_DCE_RES_IDS1_ARRAY=($TEMP_DCE_RES_IDS1)
TEMP_DCE_RES_IDS2_ARRAY=($TEMP_DCE_RES_IDS2)
TEMP_DCE_RES_IDS=("${TEMP_DCE_RES_IDS1_ARRAY[@]}" "${TEMP_DCE_RES_IDS2_ARRAY[@]}")

TEMP_FQDNS=()
for TEMP_DCE_ID in "${TEMP_DCE_RES_IDS[@]}"; do
# JSONデータを変数に保存
TEMP_JSON=$(az rest --method GET --uri $TEMP_DCE_ID?api-version=2023-03-11)
TEMP=($(echo "$TEMP_JSON" | grep -oP '(https://[a-zA-Z0-9.-]+)' | sed 's|https://||' | head -n 3))
TEMP_FQDNS+=("${TEMP[@]}")
done

echo "${TEMP_FQDNS[@]}"

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_HUB}"

# Azure Firewall に FQDN 穴あけ
TEMP_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
TEMP_FWP_NAME="fw-hub-${TEMP_LOCATION_PREFIX}-fwp"
TEMP_SPOKE_ADDRESS="${IP_SPOKE_E_PREFIXS[$i]}.0.0/16"
az network firewall policy rule-collection-group collection rule add \
--resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "AdditionalApplicationRuleCollectionGroup" \
--collection-name "Spoke E AKS" --rule-type ApplicationRule \
--name "AzureManagedPrometheus" \
--target-fqdns ${TEMP_FQDNS[@]} \
--source-addresses ${TEMP_SPOKE_ADDRESS} --protocols Https=443

done #i

```
