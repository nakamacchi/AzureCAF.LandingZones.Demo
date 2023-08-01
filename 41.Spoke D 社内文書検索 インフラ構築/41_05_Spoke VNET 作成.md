# Spoke VNET 作成

Spoke VNET を作成します。

- Spoke VNET は本番システムへのアクセスを行うための VNET 環境です。このために、以下のような VNET を作成します。
  - サブネット
    - Default :（今回は使いませんが、テスト用の VM などを立てたりするために利用してください）
    - PrivateEndpointSubnet : 各種のプライベートエンドポイントを作成するためのサブネット
    - AppServiceBackendSubnet : App Service の VNET 統合を行うためのサブネット
    - ContainerAppsSubnet : Container Apps の組み込みに利用するサブネット（今回は使いません）
  - ピアリング
    - 本番系 NW （Hub VNET）に対してピアリング
  - 名前解決
    - 本番系 NW の Azure Firewall を DNS プロキシとして利用
    - プライベートエンドポイントのプライベートゾーンは Hub VNET で管理
  - UDR・NSG
    - Hub VNET の Azure Firewall を通るように構成

```bash

# 業務システム D チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
az group create --name ${TEMP_RG_NAME} --location ${TEMP_LOCATION_NAME}
done # TEMP_LOCATION

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# Spoke D アプリ用 VNET 作成
TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_IP_PREFIX=${IP_SPOKE_D_PREFIXS[$i]}
TEMP_VNET_ADDRESS="${TEMP_IP_PREFIX}.0.0/17"
TEMP_SUBNET_DEFAULT="${TEMP_IP_PREFIX}.0.0/24"
TEMP_SUBNET_PE="${TEMP_IP_PREFIX}.1.0/24"
TEMP_SUBNET_ASB="${TEMP_IP_PREFIX}.2.0/24"
TEMP_SUBNET_CA="${TEMP_IP_PREFIX}.4.0/23"
TEMP_NSG_NAME="${TEMP_VNET_NAME}-nsg"
TEMP_UDR_NAME="${TEMP_VNET_NAME}-udr"

az network nsg create --name ${TEMP_NSG_NAME} --resource-group ${TEMP_RG_NAME}
az network route-table create --resource-group ${TEMP_RG_NAME} --name ${TEMP_UDR_NAME}
az network vnet create --resource-group ${TEMP_RG_NAME} --name ${TEMP_VNET_NAME} --address-prefixes ${TEMP_VNET_ADDRESS}
az network vnet subnet create --name "DefaultSubnet" --address-prefix ${TEMP_SUBNET_DEFAULT} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME} --route-table ${TEMP_UDR_NAME}
az network vnet subnet create --name "PrivateEndpointSubnet" --address-prefix ${TEMP_SUBNET_PE} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME} --route-table ${TEMP_UDR_NAME}
az network vnet subnet create --name "AppServiceBackendSubnet" --address-prefix ${TEMP_SUBNET_ASB} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME} --route-table ${TEMP_UDR_NAME}
az network vnet subnet create --name "ContainerAppsSubnet" --address-prefix ${TEMP_SUBNET_CA} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME} --route-table ${TEMP_UDR_NAME} --service-endpoints Microsoft.ContainerRegistry

TEMP_FW_IP=$(az network firewall ip-config list -g "rg-hub-${TEMP_LOCATION_PREFIX}" -f "fw-hub-${TEMP_LOCATION_PREFIX}" --query "[0].privateIpAddress" --output tsv --subscription ${SUBSCRIPTION_NAME_HUB})

az network route-table route create --resource-group ${TEMP_RG_NAME} --name default --route-table-name ${TEMP_UDR_NAME} --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address ${TEMP_FW_IP}

# VNET Peering HUB-SPOKED
TEMP_HUB_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourceGroups/rg-hub-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-hub-${TEMP_LOCATION_PREFIX}"
TEMP_SPOKE_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}"
 # Hub → Spoke
az network vnet peering create --name spoked --resource-group "rg-hub-${TEMP_LOCATION_PREFIX}" --vnet-name "vnet-hub-${TEMP_LOCATION_PREFIX}" --remote-vnet $TEMP_SPOKE_VNET_ID --allow-vnet-access --subscription "${SUBSCRIPTION_NAME_HUB}"
# Spoke → Hub
az network vnet peering create --name hub --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --remote-vnet $TEMP_HUB_VNET_ID --allow-vnet-access --subscription "${SUBSCRIPTION_NAME_SPOKE_D}"

# DNS サーバ切り替え
az network vnet update --name ${TEMP_VNET_NAME} --resource-group ${TEMP_RG_NAME} --dns-servers ${TEMP_FW_IP}

done

```
