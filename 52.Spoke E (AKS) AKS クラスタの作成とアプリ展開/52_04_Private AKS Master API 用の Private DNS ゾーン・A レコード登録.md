# Private AKS Master API 用の Private DNS ゾーン・A レコード登録

- VNET の DNS 名前解決が Hub DNS に向いているが、自動登録される DNS は Spoke E VNET につながってしまっている
- このため、改めて Hub VNET 側にプライベートエンドポイントの DNS を登録し、AKS クラスタをアップデートする

- （参考）AKS VMSS の起動カスタムスクリプトの中に nslookup -timeout=15 -retry=0 <APIサーバ> というコードがあり、これがエラーとなって VMSS が Failed 状態になるが、実態としては正しく稼働するので、このまま放置（DNS が Spoke E のものを利用するとエラーが出ない、原因不明）
  - https://jpaztech.github.io/blog/containers/aks-vmsscse/#ExitCode-52

```bash

# 業務システム E チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokee_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokee_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"
TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_AKS_CLUSTER_NAME="aks-spokee-${TEMP_LOCATION_PREFIX}"

# AKS クラスターのプライベート FQDN の取得
TEMP_PRIVATE_FQDN=$(az aks show --resource-group ${TEMP_RG_NAME} --name ${TEMP_AKS_CLUSTER_NAME} --query "privateFqdn" -o tsv)
# プライベート FQDN をゾーン部分と A レコード部分に分解
TEMP_RECORDSET_NAME=$(echo $TEMP_PRIVATE_FQDN | cut -d'.' -f1)
TEMP_PRIVATE_DNS_ZONE=$(echo $TEMP_PRIVATE_FQDN | cut -d'.' -f2-)

# プライベートエンドポイントのネットワークインターフェイス ID の取得
TEMP_NIC_ID=$(az network private-endpoint list --resource-group "MC_${TEMP_RG_NAME}_${TEMP_AKS_CLUSTER_NAME}_japaneast" --query "[0].networkInterfaces[0].id" -o tsv)
# ネットワークインターフェイスの詳細情報の取得
TEMP_NIC_INFO=$(az network nic show --ids $TEMP_NIC_ID)
# プライベート IP アドレスの取得
TEMP_PRIVATE_IP_ADDRESS=$(echo $TEMP_NIC_INFO | grep -oP '"privateIPAddress":\s*"\K[^"]+')

echo "Private FQDN: $TEMP_PRIVATE_FQDN"
echo "Private DNS Zone: $TEMP_PRIVATE_DNS_ZONE"
echo "Record Set Name: $TEMP_RECORDSET_NAME"
echo "Private IP Address: $TEMP_PRIVATE_IP_ADDRESS"

TEMP_PRIVATE_DNS_ZONE_NAMES="${TEMP_PRIVATE_DNS_ZONE}"

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ハブサブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_HUB}"

# Hub 側
TEMP_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
for TEMP_PRIVATE_DNS_ZONE_NAME in ${TEMP_PRIVATE_DNS_ZONE_NAMES}; do
az network private-dns zone create --resource-group ${TEMP_RG_NAME} --name ${TEMP_PRIVATE_DNS_ZONE_NAME}
# Hub VNET へリンク
TEMP_VNET_NAME="vnet-hub-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourceGroups/rg-hub-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}"
az network private-dns link vnet create --resource-group $TEMP_RG_NAME --zone-name $TEMP_PRIVATE_DNS_ZONE_NAME --name $TEMP_VNET_NAME --virtual-network $TEMP_VNET_ID --registration-enabled false

# Aレコード登録
az network private-dns record-set a add-record --resource-group ${TEMP_RG_NAME} --zone-name ${TEMP_PRIVATE_DNS_ZONE} --record-set-name ${TEMP_RECORDSET_NAME} --ipv4-address ${TEMP_PRIVATE_IP_ADDRESS}

done # TEMP_PRIVATE_DNS_ZONE_NAME

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# 管理サブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_MGMT}"

# Ops 側
TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
for TEMP_PRIVATE_DNS_ZONE_NAME in ${TEMP_PRIVATE_DNS_ZONE_NAMES}; do
az network private-dns zone create --resource-group ${TEMP_RG_NAME} --name ${TEMP_PRIVATE_DNS_ZONE_NAME}
# Ops VNET へリンク
TEMP_VNET_NAME="vnet-ops-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-ops-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}"
az network private-dns link vnet create --resource-group $TEMP_RG_NAME --zone-name $TEMP_PRIVATE_DNS_ZONE_NAME --name $TEMP_VNET_NAME --virtual-network $TEMP_VNET_ID --registration-enabled false

# Aレコード登録
az network private-dns record-set a add-record --resource-group ${TEMP_RG_NAME} --zone-name ${TEMP_PRIVATE_DNS_ZONE} --record-set-name ${TEMP_RECORDSET_NAME} --ipv4-address ${TEMP_PRIVATE_IP_ADDRESS}

done # TEMP_PRIVATE_DNS_ZONE_NAME

# 業務システム E チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokee_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokee_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_AKS_CLUSTER_NAME="aks-spokee-${TEMP_LOCATION_PREFIX}"

# AKS に Private Endpoint を再認識させる
echo "Updating AKS cluster..."
az resource update --resource-group ${TEMP_RG_NAME} --name ${TEMP_AKS_CLUSTER_NAME} --namespace Microsoft.ContainerService --resource-type ManagedClusters

done # TEMP_LOCATION

```
