# メンテ VM から ACR へのアクセス権の付与

```bash

# VM の Managed ID でプッシュできるように再構成

# 業務システム E チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokee_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokee_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

# Spoke E サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

# VM からのプッシュアクセス権限の付与 
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_ACR_NAME="acrspokee${UNIQUE_SUFFIX}${TEMP_LOCATION_PREFIX}"

# VM からのプッシュを可能に
TEMP_VM_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourceGroups/rg-spokeemtn-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Compute/virtualMachines/vm-mtn-${TEMP_LOCATION_PREFIX}"
TEMP_IDENTITY_ID=$(az vm show --id ${TEMP_VM_ID} --query identity.principalId -o tsv)
TEMP_ACR_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourceGroups/rg-spokee-${TEMP_LOCATION_PREFIX}/providers/Microsoft.ContainerRegistry/registries/${TEMP_ACR_NAME}"
az role assignment create --assignee $TEMP_IDENTITY_ID --role "AcrPush" --scope $TEMP_ACR_ID
az role assignment create --assignee $TEMP_IDENTITY_ID --role "AcrPull" --scope $TEMP_ACR_ID

done #TEMP_LOCATION

```
