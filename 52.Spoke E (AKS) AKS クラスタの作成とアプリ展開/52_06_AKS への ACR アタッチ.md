# AKS への ACR アタッチ

```bash

# https://learn.microsoft.com/ja-jp/azure/aks/cluster-container-registry-integration?toc=%2Fazure%2Fcontainer-registry%2Ftoc.json&bc=%2Fazure%2Fcontainer-registry%2Fbreadcrumb%2Ftoc.json&tabs=azure-cli#attach-an-acr-to-an-existing-aks-cluster

# AKS クラスタに ACR へのアクセス権限を Azure RBAC で割り当てる
# 内部的には Agent Pool に対してユーザ割り当て MID が作成され、AcrPull 権限が付与される

# 業務システム E チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokee_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokee_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_AKS_CLUSTER_NAME="aks-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_ACR_NAME="acrspokee${UNIQUE_SUFFIX}${TEMP_LOCATION_PREFIX}"

TEMP_ACR_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourceGroups/rg-spokee-${TEMP_LOCATION_PREFIX}/providers/Microsoft.ContainerRegistry/registries/${TEMP_ACR_NAME}"

az aks update --name ${TEMP_AKS_CLUSTER_NAME} --resource-group ${TEMP_RG_NAME} --attach-acr ${TEMP_ACR_ID}

done #i

```
