# Advanced Network Observability の有効化

- https://learn.microsoft.com/en-us/azure/aks/advanced-network-observability-cli?tabs=non-cilium

```bash
# Install the aks-preview extension
az extension add --name aks-preview
# Update the extension to make sure you have the latest version installed
az extension update --name aks-preview
```

```bash

# 業務システム E チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokee_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokee_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

# Preview 機能の有効化
TEMP_TARGET_SUBSCRIPTION_IDS=$SUBSCRIPTION_ID_SPOKE_E

TEMP_RP_NAMES="Microsoft.ContainerService"
TEMP_FEATURE_NAMES="\
Microsoft.ContainerService,AdvancedNetworkingPreview \
"
 
for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
echo "Initialize subscription... ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"
 
# RP 有効化
for TEMP_RP_NAME in $TEMP_RP_NAMES; do
echo "Registering ${TEMP_RP_NAME} RP on ${TEMP_SUBSCRIPTION_ID}..."
az provider register --namespace "${TEMP_RP_NAME}"
done #TEMP_RP_NAME

# Feature 有効化
for TEMP_FEATURE_NAME_TEMP in $TEMP_FEATURE_NAMES; do
  # 分解して利用
  TEMP=(${TEMP_FEATURE_NAME_TEMP//,/ })
  TEMP_FEATURE_NAMESPACE=${TEMP[0]}
  TEMP_FEATURE_NAME=${TEMP[1]}
az feature register --namespace ${TEMP_FEATURE_NAMESPACE} --name ${TEMP_FEATURE_NAME}
done #TEMP_FEATURE_NAME_TEMP

done #TEMP_SUBSCRIPTION_ID
 
for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
echo "Waiting initializing subscription... ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"

# RP 有効化待ち
for TEMP_RP_NAME in $TEMP_RP_NAMES; do
while [ $(az provider show --namespace "${TEMP_RP_NAME}" --query registrationState -o tsv) != "Registered" ]
do
  echo "$(az provider show --namespace "${TEMP_RP_NAME}" --query registrationState -o tsv) on ${TEMP_SUBSCRIPTION_ID} ${TEMP_RP_NAME}..."
  sleep 10
done
done #TEMP_RP_NAME

# Feature 有効化待ち
for TEMP_FEATURE_NAME_TEMP in $TEMP_FEATURE_NAMES; do
  # 分解して利用
  TEMP=(${TEMP_FEATURE_NAME_TEMP//,/ })
  TEMP_FEATURE_NAMESPACE=${TEMP[0]}
  TEMP_FEATURE_NAME=${TEMP[1]}
while [ $(az feature show --namespace ${TEMP_FEATURE_NAMESPACE} --name ${TEMP_FEATURE_NAME} --query properties.state -o tsv) != "Registered" ]
do
  echo "$(az feature show --namespace ${TEMP_FEATURE_NAMESPACE} --name ${TEMP_FEATURE_NAME} --query properties.state -o tsv) ${TEMP_FEATURE_NAMESPACE}/${TEMP_FEATURE_NAME} ..."
  sleep 10
done
done #TEMP_FEATURE_NAME_TEMP

done #TEMP_SUBSCRIPTION_ID

```

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_AKS_CLUSTER_NAME="aks-spokee-${TEMP_LOCATION_PREFIX}"

az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"
az aks update --resource-group ${TEMP_RG_NAME} --name ${TEMP_AKS_CLUSTER_NAME} --enable-advanced-network-observability

done #i

```
