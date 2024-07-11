

```bash

# CAE DNS 登録

# 業務システム D チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# Spoke D サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_CAE_NAME="cae-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_CA_NAME="ca-spoked-${TEMP_LOCATION_PREFIX}"

TEMP_CAE_DEFAULT_DOMAIN=$(az containerapp env show --name ${TEMP_CAE_NAME} --resource-group ${TEMP_RG_NAME} --query properties.defaultDomain -o tsv)
TEMP_CAE_STATIC_IP=$(az containerapp env show --name ${TEMP_CAE_NAME} --resource-group ${TEMP_RG_NAME} --query properties.staticIp -o tsv)

# Hub にプライベート DNS ゾーンとして登録する
# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"
 
# プライベート DNS ゾーンを登録するサブスクリプション ID とリソースグループ、リンク先 VNET
TEMP_PDZ_SUBSCRIPTION_ID="${SUBSCRIPTION_ID_HUB}"
TEMP_PDZ_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
TEMP_PDZ_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourceGroups/rg-hub-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-hub-${TEMP_LOCATION_PREFIX}"

# Private DNS Zone 作成
TEMP_REQUIRED_ZONE_NAME=${TEMP_CAE_DEFAULT_DOMAIN}
echo "Create DNS Zones on ${TEMP_PDZ_SUBSCRIPTION_ID} ${TEMP_PDZ_RG_NAME} : ${TEMP_REQUIRED_ZONE_NAME}"

TEMP_PRIVATE_DNS_ZONE_ID="/subscriptions/${TEMP_PDZ_SUBSCRIPTION_ID}/resourceGroups/${TEMP_PDZ_RG_NAME}/providers/Microsoft.Network/privateDnsZones/${TEMP_REQUIRED_ZONE_NAME}"

TEMP_ISEXISTS=$(az rest --method get --uri "${TEMP_PRIVATE_DNS_ZONE_ID}?api-version=2020-06-01" --query id -o tsv)
if [[ $TEMP_ISEXISTS == *"ERROR"* || -z $TEMP_ISEXISTS ]]; then
  echo "Private DNS Zone ${TEMP_REQUIRED_ZONE_NAME} does not exist. Creating Private DNS Zone on Subscription ${TEMP_PDZ_SUBSCRIPTION_ID}."
  az network private-dns zone create --resource-group ${TEMP_PDZ_RG_NAME} --name ${TEMP_REQUIRED_ZONE_NAME} --subscription "${TEMP_PDZ_SUBSCRIPTION_ID}"
  echo "Linking Private DNS Zones ${TEMP_REQUIRED_ZONE_NAME} to VNET ${TEMP_PDZ_VNET_ID}."
  TEMP_PDZ_VNET_NAME=${TEMP_PDZ_VNET_ID##*/}
  az network private-dns link vnet create --resource-group $TEMP_PDZ_RG_NAME --zone-name $TEMP_REQUIRED_ZONE_NAME --name $TEMP_PDZ_VNET_NAME --virtual-network $TEMP_PDZ_VNET_ID --registration-enabled false --subscription "${TEMP_PDZ_SUBSCRIPTION_ID}"
else
  echo "Private DNS Zone already exists."
fi

# IP アドレスを登録する
az network private-dns record-set a add-record --resource-group ${TEMP_PDZ_RG_NAME} --zone-name ${TEMP_REQUIRED_ZONE_NAME} --record-set-name "*" --ipv4-address ${TEMP_CAE_STATIC_IP} --subscription "${TEMP_PDZ_SUBSCRIPTION_ID}"

done # TEMP_LOCATION

```

```bash

if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
# Spoke D サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_CAE_NAME="cae-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_CA_NAME="ca-spoked-${TEMP_LOCATION_PREFIX}"

TEMP_CAE_DEFAULT_DOMAIN=$(az containerapp env show --name ${TEMP_CAE_NAME} --resource-group ${TEMP_RG_NAME} --query properties.defaultDomain -o tsv)

cat <<EOF
# 以下の URL に本番系 NW（Hub 側）の端末からアクセスを試みてください。
https://${TEMP_CA_NAME}.${TEMP_CAE_DEFAULT_DOMAIN}/
EOF

done # TEMP_LOCATION

```