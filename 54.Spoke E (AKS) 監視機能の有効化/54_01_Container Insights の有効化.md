# Container Insights の有効化

```bash

# Spoke E 作業用アカウントだと LAW に対するキー取得権限が不足し、デプロイメントに失敗する。
# Microsoft.OperationalInsights/workspaces/sharedKeys/action
# このため gov_change を利用する。

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# Container Insights の有効化
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_AKS_CLUSTER_NAME="aks-spokee-${TEMP_LOCATION_PREFIX}"

TEMP_AKS_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.ContainerService/managedClusters/${TEMP_AKS_CLUSTER_NAME}"

TEMP_LAW_RES_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.OperationalInsights/workspaces/law-vdc-${TEMP_LOCATION_PREFIX}"

az aks enable-addons --resource-group ${TEMP_RG_NAME} --name ${TEMP_AKS_CLUSTER_NAME} --addons monitoring --workspace-resource-id $TEMP_LAW_RES_ID

done #i

```
