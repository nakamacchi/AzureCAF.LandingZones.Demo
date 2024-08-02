# DevBox 用ネットワーク環境の作成

DevBox を利用するには、DevBox を収容するためのネットワーク環境を作成する必要があります。下図のようなネットワーク（Hub-Spoke 構造）を作成します。なお今回の構成では、PoC 用途の DevBox 環境を作成するため、オンプレミスとの接続は行いません（＝ハブサブスクリプションのハブとは別に、開発用のハブを作成します）。

![picture 9](./images/329073a138ffd0a2e68ae23dba44f7d8a2575629c11ab0407bc787da3f8acf28.png)  

下記のスクリプトにより、以下の作業を行います。

- devhub, devbox の VNET の作成とピアリング
- Azure Firewall の作成と UDR の設定（※ 通信ルール設定は次セクション）

なお、ネットワークを設計する際、ひとつのサブネットに複数の開発ボックスプールが割り当てられる場合があることに注意してください。（一つの開発プロジェクトで異なるサイズの VM を利用する場合などに発生します）　このため本サンプルでは、サブネット名に開発プロジェクト名を利用し、当該サブネットには当該開発プロジェクトの複数の開発ボックスプールが割り当てられる、という想定で設計しています。

```bash

IP_DEV1_HUB_PREFIXS[0]="10.30"
IP_DEV1_HUB_PREFIXS[1]="10.80"
IP_DEV1_DEVBOX_PREFIXS[0]="10.31"
IP_DEV1_DEVBOX_PREFIXS[1]="10.81"

if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_dev1_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_dev1_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# Dev1 サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_DEV1}"

# メインリージョンに開発環境を作成する
TEMP_LOCATION_NAME=${LOCATION_NAMES[0]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}
TEMP_IP_DEV1_HUB_PREFIX=${IP_DEV1_HUB_PREFIXS[0]}
TEMP_IP_DEV1_DEVBOX_PREFIX=${IP_DEV1_DEVBOX_PREFIXS[0]}

#########################################

# 開発用 Hub の作成
TEMP_IP_PREFIX=${TEMP_IP_DEV1_HUB_PREFIX}

TEMP_RG_NAME="rg-devhub-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-devhub-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_ADDRESS="${TEMP_IP_PREFIX}.0.0/16"
TEMP_NSG_NAME="${TEMP_VNET_NAME}-nsg"
TEMP_UDR_NAME="${TEMP_VNET_NAME}-udr"
TEMP_SUBNET_FW="${TEMP_IP_PREFIX}.251.0/25"
TEMP_SUBNET_FWMGMT="${TEMP_IP_PREFIX}.251.128/25"

az group create --name ${TEMP_RG_NAME} --location ${TEMP_LOCATION_NAME}
az network vnet create --resource-group ${TEMP_RG_NAME} --name ${TEMP_VNET_NAME} --address-prefixes ${TEMP_VNET_ADDRESS}

az network vnet subnet create --name "AzureFirewallSubnet" --address-prefix ${TEMP_SUBNET_FW} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME}
az network vnet subnet create --name "AzureFirewallManagementSubnet" --address-prefix ${TEMP_SUBNET_FWMGMT} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME}

# DevBox 用 Spoke の作成
TEMP_IP_PREFIX=${TEMP_IP_DEV1_DEVBOX_PREFIX}

TEMP_RG_NAME="rg-devbox-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-devbox-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_ADDRESS="${TEMP_IP_PREFIX}.0.0/16"
TEMP_NSG_NAME="${TEMP_VNET_NAME}-nsg"
TEMP_UDR_NAME="${TEMP_VNET_NAME}-udr"
TEMP_SUBNET_DEVPROJECT_X="${TEMP_IP_PREFIX}.10.0/24"
TEMP_SUBNET_DEVPROJECT_Y="${TEMP_IP_PREFIX}.20.0/24"

az group create --name ${TEMP_RG_NAME} --location ${TEMP_LOCATION_NAME}
az network vnet create --resource-group ${TEMP_RG_NAME} --name ${TEMP_VNET_NAME} --address-prefixes ${TEMP_VNET_ADDRESS}
az network route-table create --resource-group ${TEMP_RG_NAME} --name ${TEMP_UDR_NAME}
az network nsg create --name ${TEMP_NSG_NAME} --resource-group ${TEMP_RG_NAME}

az network vnet subnet create --name "DevProjectXSubnet" --address-prefix ${TEMP_SUBNET_DEVPROJECT_X} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME} --route-table ${TEMP_UDR_NAME}
az network vnet subnet create --name "DevProjectYSubnet" --address-prefix ${TEMP_SUBNET_DEVPROJECT_Y} --resource-group ${TEMP_RG_NAME} --vnet-name ${TEMP_VNET_NAME} --nsg ${TEMP_NSG_NAME} --route-table ${TEMP_UDR_NAME}

# VNET ピアリング設定
TEMP_HUB_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devhub-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-devhub-${TEMP_LOCATION_PREFIX}"
TEMP_SPOKE_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devbox-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-devbox-${TEMP_LOCATION_PREFIX}"

# Hub → Spoke
az rest --method PUT --uri "${TEMP_HUB_VNET_ID}/virtualNetworkPeerings/devbox?syncRemoteAddressSpace=true&api-version=2023-05-01" --body @- << EOF
{
  "properties": {
    "allowVirtualNetworkAccess": true,
    "allowForwardedTraffic": true,
    "allowGatewayTransit": true,
    "useRemoteGateways": false,
    "remoteVirtualNetwork": {
      "id": "${TEMP_SPOKE_VNET_ID}"
    }
  }
}
EOF

# Spoke → Hub
az rest --method PUT --uri "${TEMP_SPOKE_VNET_ID}/virtualNetworkPeerings/devhub?syncRemoteAddressSpace=true&api-version=2023-05-01" --body @- << EOF
{
  "properties": {
    "allowVirtualNetworkAccess": true,
    "allowForwardedTraffic": false,
    "allowGatewayTransit": false,
    "useRemoteGateways": false,
    "remoteVirtualNetwork": {
      "id": "${TEMP_HUB_VNET_ID}"
    }
  }
}
EOF

# Azure Firewall 作成とルートテーブルの設定
TEMP_RG_NAME="rg-devhub-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-devhub-${TEMP_LOCATION_PREFIX}"
TEMP_FW_NAME="fw-devhub-${TEMP_LOCATION_PREFIX}"
TEMP_FW_PIP_NAME="${TEMP_FW_NAME}-pip"
TEMP_FW_MGMT_PIP_NAME="${TEMP_FW_NAME}-mgmt-pip"
TEMP_FWP_NAME="${TEMP_FW_NAME}-fwp"
TEMP_FW_SKU="Standard" # DNS proxy 機能を利用するため Standard が必要

# Firewall Policy 作成
az network firewall policy create --name ${TEMP_FWP_NAME} --resource-group ${TEMP_RG_NAME} --sku Standard

# パブリック IP、管理 IP を作成
az network public-ip create --name ${TEMP_FW_PIP_NAME} --resource-group ${TEMP_RG_NAME} --location ${TEMP_LOCATION_NAME} --allocation-method static --sku standard
az network public-ip create --name ${TEMP_FW_MGMT_PIP_NAME} --resource-group ${TEMP_RG_NAME} --location ${TEMP_LOCATION_NAME} --allocation-method static --sku standard

# Azure Firewall 作成
az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/azureFirewalls/${TEMP_FW_NAME}?api-version=2023-05-01" --body @- <<EOF
{
  "location": "${TEMP_LOCATION_NAME}",
  "properties": {
    "ipConfigurations": [
      {
        "name": "${TEMP_FW_PIP_NAME}",
        "properties": {
          "subnet": {
            "id": "$(az network vnet subnet show --name AzureFirewallSubnet --vnet-name ${TEMP_VNET_NAME} --resource-group ${TEMP_RG_NAME} --query id -o tsv)"
          },
          "publicIpAddress": {
            "id": "$(az network public-ip show --name ${TEMP_FW_PIP_NAME} --resource-group ${TEMP_RG_NAME} --query id -o tsv)"
          }
        }
      }
    ],
    "sku": {
      "tier": "Standard"
    },
    "managementIpConfiguration": {
      "name": "${TEMP_FW_MGMT_PIP_NAME}",
      "properties": {
        "subnet": {
          "id": "$(az network vnet subnet show --name AzureFirewallManagementSubnet --vnet-name ${TEMP_VNET_NAME} --resource-group ${TEMP_RG_NAME} --query id -o tsv)"
        },
        "publicIPAddress": {
          "id": "$(az network public-ip show --name ${TEMP_FW_MGMT_PIP_NAME} --resource-group ${TEMP_RG_NAME} --query id -o tsv)"
        }
      }
    },
    "firewallPolicy": {
      "id": "$(az network firewall policy show --name "${TEMP_FWP_NAME}" --resource-group "${TEMP_RG_NAME}" --query id -o tsv)"
    }
  }
}
EOF

# Azure Firewall のプロビジョニングが完了していないと IP アドレスが取得できないので待機
while true
do
  STATE=$(az network firewall show -g ${TEMP_RG_NAME} -n ${TEMP_FW_NAME} --query provisioningState -o tsv)
  echo "Waiting for provisioning... ${STATE}"
  if [ $STATE == "Succeeded" ]; then
    break
  fi
  sleep 10
done

TEMP_FW_IP=$(az network firewall ip-config list -g ${TEMP_RG_NAME} -f ${TEMP_FW_NAME} --query "[0].privateIpAddress" --output tsv)

# DevBox のルートテーブルを更新
az network route-table route create --resource-group "rg-devbox-${TEMP_LOCATION_PREFIX}" --route-table-name "vnet-devbox-${TEMP_LOCATION_PREFIX}-udr" --name default --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address ${TEMP_FW_IP}

# VNET の DNS を Azure Firewall に向ける（ネットワークルールの利用のため）
az network vnet update --resource-group "rg-devbox-${TEMP_LOCATION_PREFIX}" --name "vnet-devbox-${TEMP_LOCATION_PREFIX}" --dns-servers ${TEMP_FW_IP}
az network vnet update --resource-group "rg-devhub-${TEMP_LOCATION_PREFIX}" --name "vnet-devhub-${TEMP_LOCATION_PREFIX}" --dns-servers ${TEMP_FW_IP}

```
