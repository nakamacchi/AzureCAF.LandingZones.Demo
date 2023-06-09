# Ops VNET 作成

```bash
 
# 共通基盤管理チーム／① 初期構築時の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_plat_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# 運用管理基盤の作成
az account set -s "${SUBSCRIPTION_ID_MGMT}"
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
  # Ops VNET 作成
  TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
  TEMP_VNET_NAME="vnet-ops-${TEMP_LOCATION_PREFIX}"
  TEMP_IP_PREFIX=${IP_OPS_PREFIXS[$i]}
  TEMP_VNET_ADDRESS="${TEMP_IP_PREFIX}.0.0/16"
  TEMP_SUBNET_DEFAULT="${TEMP_IP_PREFIX}.0.0/24"
  TEMP_SUBNET_PE="${TEMP_IP_PREFIX}.250.0/24"
  TEMP_SUBNET_FW="${TEMP_IP_PREFIX}.251.0/25"
  TEMP_SUBNET_FWMGMT="${TEMP_IP_PREFIX}.251.128/25"
  TEMP_SUBNET_BASTION="${TEMP_IP_PREFIX}.252.0/24"
  TEMP_SUBNET_PROXY="${TEMP_IP_PREFIX}.254.0/24"
  TEMP_NSG_NAME="${TEMP_VNET_NAME}-nsg"
 
  az group create --name ${TEMP_RG_NAME} --location ${TEMP_LOCATION_NAME}
  az network nsg create --name ${TEMP_NSG_NAME} --resource-group ${TEMP_RG_NAME}
  az network vnet create --resource-group ${TEMP_RG_NAME} --name ${TEMP_VNET_NAME} --address-prefixes ${TEMP_VNET_ADDRESS}
  az network vnet subnet create --name "DefaultSubnet" --address-prefix ${TEMP_SUBNET_DEFAULT} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME}
  az network vnet subnet create --name "AzureFirewallSubnet" --address-prefix ${TEMP_SUBNET_FW} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME}
  az network vnet subnet create --name "AzureFirewallManagementSubnet" --address-prefix ${TEMP_SUBNET_FWMGMT} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME}
  az network vnet subnet create --name "PrivateEndpointSubnet" --address-prefix ${TEMP_SUBNET_PE} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME}
  az network vnet subnet create --name "AzureBastionSubnet" --address-prefix ${TEMP_SUBNET_BASTION} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME}
  az network vnet subnet create --name "ProxySubnet" --address-prefix ${TEMP_SUBNET_PROXY} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME}
 
done
 
# Bastion 作成
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
  # Ops Bastion 作成（インフラ管理用 VM へのログイン用踏み台）
  # ※ 管理端末を Azure VM として持つ必要がなければ不要
  TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
  TEMP_BASTION_NAME="bst-ops-${TEMP_LOCATION_PREFIX}"
  TEMP_BASTION_PIP_NAME="${TEMP_BASTION_NAME}-pip"
  TEMP_VNET_NAME="vnet-ops-${TEMP_LOCATION_PREFIX}"
 
  az network public-ip create --name ${TEMP_BASTION_PIP_NAME} --resource-group ${TEMP_RG_NAME} --sku Standard
  az network bastion create --name ${TEMP_BASTION_NAME} --public-ip-address ${TEMP_BASTION_PIP_NAME} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --location ${TEMP_LOCATION_NAME} --no-wait
 
done
 
```
