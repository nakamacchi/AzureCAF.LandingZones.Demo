# Spoke VNET 作成

```bash

# ピアリングは双方のネットワークの可視化と書き込み権限が必要
# このため DC 全体の Network Contributor ロールでの作業を行う
 
# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# Spoke A サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_A}"
 
# スポーク A VNET の作成
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
  # Spoe A VNET 作成
  TEMP_RG_NAME="rg-spokea-${TEMP_LOCATION_PREFIX}"
  TEMP_VNET_NAME="vnet-spokea-${TEMP_LOCATION_PREFIX}"
  TEMP_IP_PREFIX=${IP_SPOKE_A_PREFIXS[$i]}
  TEMP_VNET_ADDRESS="${TEMP_IP_PREFIX}.0.0/16"
  TEMP_SUBNET_WEB="${TEMP_IP_PREFIX}.10.0/24"
  TEMP_SUBNET_DB="${TEMP_IP_PREFIX}.20.0/24"
  TEMP_SUBNET_DEFAULT="${TEMP_IP_PREFIX}.0.0/24"
  TEMP_NSG_NAME="${TEMP_VNET_NAME}-nsg"
  TEMP_UDR_NAME="${TEMP_VNET_NAME}-udr"
 
  az network nsg create --name ${TEMP_NSG_NAME} --resource-group ${TEMP_RG_NAME}
  az network vnet create --resource-group ${TEMP_RG_NAME} --name ${TEMP_VNET_NAME} --address-prefixes ${TEMP_VNET_ADDRESS}
  az network vnet subnet create --name "WebSubnet" --address-prefix ${TEMP_SUBNET_WEB} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME}
  az network vnet subnet create --name "DbSubnet" --address-prefix ${TEMP_SUBNET_DB} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME}
  az network vnet subnet create --name "DefaultSubnet" --address-prefix ${TEMP_SUBNET_DEFAULT} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME}
 
  # UDR 作成
  az network route-table create --resource-group ${TEMP_RG_NAME} --name ${TEMP_UDR_NAME}
 
  az account set -s "${SUBSCRIPTION_NAME_HUB}"
  TEMP_FW_IP=$(az network firewall ip-config list -g "rg-hub-${TEMP_LOCATION_PREFIX}" -f "fw-hub-${TEMP_LOCATION_PREFIX}" --query "[0].privateIpAddress" --output tsv)
  az account set -s "${SUBSCRIPTION_NAME_SPOKE_A}"
 
  az network route-table route create --resource-group ${TEMP_RG_NAME} --name default --route-table-name ${TEMP_UDR_NAME} --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address ${TEMP_FW_IP}
 
  # UDR 割り当て
  az network vnet subnet update --resource-group ${TEMP_RG_NAME} --route-table ${TEMP_UDR_NAME} --ids $(az network vnet subnet show --resource-group ${TEMP_RG_NAME} --vnet-name $TEMP_VNET_NAME --name "WebSubnet" --query id -o tsv)
  az network vnet subnet update --resource-group ${TEMP_RG_NAME} --route-table ${TEMP_UDR_NAME} --ids $(az network vnet subnet show --resource-group ${TEMP_RG_NAME} --vnet-name $TEMP_VNET_NAME --name "DbSubnet" --query id -o tsv)
  az network vnet subnet update --resource-group ${TEMP_RG_NAME} --route-table ${TEMP_UDR_NAME} --ids $(az network vnet subnet show --resource-group ${TEMP_RG_NAME} --vnet-name $TEMP_VNET_NAME --name "DefaultSubnet" --query id -o tsv)
 
done
 
# VNET Peering の実施
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
  # Spoe A VNET 作成
  TEMP_RG_NAME="rg-spokea-${TEMP_LOCATION_PREFIX}"
  TEMP_VNET_NAME="vnet-spokea-${TEMP_LOCATION_PREFIX}"
  TEMP_IP_PREFIX=${IP_SPOKE_A_PREFIXS[$i]}
  TEMP_VNET_ADDRESS="${TEMP_IP_PREFIX}.0.0/16"
  TEMP_SUBNET_WEB="${TEMP_IP_PREFIX}.10.0/24"
  TEMP_SUBNET_DB="${TEMP_IP_PREFIX}.20.0/24"
  TEMP_NSG_NAME="${TEMP_VNET_NAME}-nsg"
  TEMP_UDR_NAME="${TEMP_VNET_NAME}-udr"
 
  # Hub VNET と接続し、オンプレ環境からのアクセス Azure Firewall による outbound アクセスができるようにする
  # VNET Peering
  TEMP_HUB_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourceGroups/rg-hub-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-hub-${TEMP_LOCATION_PREFIX}"
  TEMP_SPOKE_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_A}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}"
 
  # Hub → Spoke
  az account set -s "${SUBSCRIPTION_NAME_HUB}"
  az network vnet peering create --name spokea --resource-group "rg-hub-${TEMP_LOCATION_PREFIX}" --vnet-name "vnet-hub-${TEMP_LOCATION_PREFIX}" --remote-vnet $TEMP_SPOKE_VNET_ID --allow-vnet-access
 
  # Spoke → Hub
  az account set -s "${SUBSCRIPTION_NAME_SPOKE_A}"
  az network vnet peering create --name hub --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --remote-vnet $TEMP_HUB_VNET_ID --allow-vnet-access
 
  # 運用管理用の Ops VNET と接続し、メンテナンスできるようにする
  # VNET Peering
  TEMP_OPS_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-ops-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-ops-${TEMP_LOCATION_PREFIX}"
  TEMP_SPOKE_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_A}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}"
 
  # Ops → Spoke
  az account set -s "${SUBSCRIPTION_NAME_MGMT}"
  az network vnet peering create --name spokea --resource-group "rg-ops-${TEMP_LOCATION_PREFIX}" --vnet-name "vnet-ops-${TEMP_LOCATION_PREFIX}" --remote-vnet $TEMP_SPOKE_VNET_ID --allow-vnet-access
 
  # Spoke → Ops
  az account set -s "${SUBSCRIPTION_NAME_SPOKE_A}"
  az network vnet peering create --name ops --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --remote-vnet $TEMP_OPS_VNET_ID --allow-vnet-access
 
done

```
