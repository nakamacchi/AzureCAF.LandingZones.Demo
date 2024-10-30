# （参考）Bastion Premium によるセッション録画

運用管理 VNET 上の Bastion は標準的な Standard SKU で作成しましたが、上位 SKU の Premium で作成すると、画面の強制録画機能を利用することができます。 ([参考](https://www.kentsu.website/ja/posts/2024/bastion_recording/))

## 画面の強制録画機能の有効化

以下の作業により、画面の強制録画機能を有効化します。

- Premium SKU への変更
- 録画データを保存するための Blob ストレージの作成
- Blob ストレージの CORS 機能の有効化
- Bastion が Blob ストレージへアクセスするための SAS キーの作成
- 取得した SAS キーの Bastion への設定

有効化が済んだら、Bastion 経由で何らかの VM にログインして操作・ログオフします。その後、Azure Portal の Bastion の "Session Recordings" セクションに行くと、録画したものを参照することができます。

```bash

# 共通基盤管理チーム／① 初期構築時の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_plat_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_plat_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
 
# 運用管理基盤の作成
az account set -s "${SUBSCRIPTION_ID_MGMT}"
 
for i in ${VDC_NUMBERS}; do

TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
TEMP_BASTION_NAME="bst-ops-${TEMP_LOCATION_PREFIX}"
TEMP_BASTION_PIP_NAME="${TEMP_BASTION_NAME}-pip"
TEMP_VNET_NAME="vnet-ops-${TEMP_LOCATION_PREFIX}"

# az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/bastionHosts/${TEMP_BASTION_NAME}?api-version=2023-09-01"
az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/bastionHosts/${TEMP_BASTION_NAME}?api-version=2023-09-01" --body @- <<EOF
{
      "location": "${TEMP_LOCATION_NAME}",
      "properties": {
        "scaleUnits": 2,
        "enableTunneling": false,
        "enableIpConnect": false,
        "disableCopyPaste": false,
        "ipConfigurations": [
            {
                "id": "/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/bastionHosts/${TEMP_BASTION_NAME}/bastionHostIpConfigurations/bastion_ip_config",
                "type": "Microsoft.Network/bastionHosts/bastionHostIpConfigurations",
                "name": "bastion_ip_config",
                "properties": {
                    "privateIPAllocationMethod": "Dynamic",
                    "publicIPAddress": {
                        "id": "/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/publicIPAddresses/${TEMP_BASTION_NAME}-pip"
                    },
                    "subnet": {
                        "id": "/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}/subnets/AzureBastionSubnet"
                    }
                }
            }
        ],
        "enableKerberos": false,
        "enableShareableLink": false,
        "enableSessionRecording": true
    },
    "sku": {
        "name": "Premium"
    }
}
EOF

TEMP_BASTION_NAME="bst-ops-${TEMP_LOCATION_PREFIX}"
TEMP_RES_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/bastionHosts/${TEMP_BASTION_NAME}"
while true
do
  PROV_STATUS=$(az rest --method GET --uri "${TEMP_RES_ID}?api-version=2023-09-01" --query properties.provisioningState -o tsv)
  echo "Waiting for provisioning. Current state is $PROV_STATUS ..."
  if [ "$PROV_STATUS" == "Succeeded" ] || [ "$PROV_STATUS" == "Warning" ]; then
    break
  fi
  sleep 10
done

TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
TEMP_STORAGE_NAME="stbstops${TEMP_LOCATION_PREFIX}${UNIQUE_SUFFIX}"

az storage account create --name "${TEMP_STORAGE_NAME}" --resource-group "${TEMP_RG_NAME}" --location "${TEMP_LOCATION_NAME}" --sku Standard_RAGRS --subscription "${SUBSCRIPTION_ID_MGMT}"

TEMP_RES_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/bastionHosts/bst-ops-${TEMP_LOCATION_PREFIX}"
TEMP_BST_URL=$(az rest --method GET --uri "${TEMP_RES_ID}?api-version=2023-09-01" --query properties.dnsName -o tsv)

TEMP_STORAGE_KEY=$(az storage account keys list --account-name "${TEMP_STORAGE_NAME}" --resource-group "${TEMP_RG_NAME}" --query [0].value -o tsv)
az storage cors add --methods GET --origins "https://${TEMP_BST_URL}" --services b --max-age 86400 --account-name "${TEMP_STORAGE_NAME}" --account-key "${TEMP_STORAGE_KEY}"

TEMP_STORAGE_CONTAINER_NAME="bastionrecordings"
az storage container create --name "${TEMP_STORAGE_CONTAINER_NAME}" --account-name "${TEMP_STORAGE_NAME}" --account-key "${TEMP_STORAGE_KEY}"

# SAS キー生成
TEMP_START_DATE=`date -u -d "30 minutes ago" '+%Y-%m-%dT%H:%MZ'`
TEMP_END_DATE=`date -u -d "5 years" '+%Y-%m-%dT%H:%MZ'`
TEMP_SAS_KEY=$(az storage container generate-sas --name "${TEMP_STORAGE_CONTAINER_NAME}" --account-name "${TEMP_STORAGE_NAME}" --account-key "${TEMP_STORAGE_KEY}" --start "${TEMP_START_DATE}" --expiry "${TEMP_END_DATE}" --permissions rcwl --https-only -o tsv)
TEMP_SAS_URL="https://${TEMP_STORAGE_NAME}.blob.core.windows.net/${TEMP_STORAGE_CONTAINER_NAME}?${TEMP_SAS_KEY}"

# Bastion に設定
az rest --method POST --uri "${TEMP_RES_ID}/setsessionrecordingsasurl?api-version=2023-09-01" --body @- <<EOF
{
  "sasUrl":"${TEMP_SAS_URL}"
}
EOF

done # TEMP_LOCATION

```

## Blob ストレージの閉域化

ここまでの構成では録画データに対してパブリックエンドポイント経由でアクセスすることになりますが、Blob ストレージを閉域化したい場合には、さらに以下の作業を行います。

- ストレージアカウントのパブリックアクセスを禁止
- ストレージアカウントのプライベートエンドポイントを vnet-ops-XXX に作成
- vnet-ops-XXX 上でストレージアカウントのプライベートエンドポイントの DNS 名前解決ができるように構成

以上の作業により、録画データをインターネットから参照することができなくなります。（vm-ops-xxx 端末などからポータルサイトにアクセスして参照してください。）

※ 本サンプルでは実装しませんが、Premium SKU であれば、Bastion へのインターネットからのアクセスを禁止することもできます。この方法を採った場合には、Express Route Private Peering や VPN などで vnet-ops-XXX に入り、そこから Bastion にアクセスしてください。

```bash

# 共通基盤管理チーム／① 初期構築時の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_plat_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_plat_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
 
# 運用管理基盤の作成
az account set -s "${SUBSCRIPTION_ID_MGMT}"
 
# Private Endpoint 化
TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
TEMP_STORAGE_NAME="stbstops${TEMP_LOCATION_PREFIX}${UNIQUE_SUFFIX}"
az storage account update --name "${TEMP_STORAGE_NAME}" --resource-group "${TEMP_RG_NAME}" --subscription "${SUBSCRIPTION_ID_MGMT}" --public-network-access Disabled

############################################
# ストレージアカウントのプライベートエンドポイントを vnet-ops-XXX に作成

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# PE を作成するリソースの定義
TEMP_STORAGE_NAME="stbstops${TEMP_LOCATION_PREFIX}${UNIQUE_SUFFIX}"
TEMP_RESOURCE_IDS="\
/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-ops-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Storage/storageAccounts/${TEMP_STORAGE_NAME}
"

# PE を作成する VNET/Subnet
TEMP_PE_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
TEMP_PE_VNET_NAME="vnet-ops-${TEMP_LOCATION_PREFIX}"
TEMP_PE_SUBNET_NAME="PrivateEndpointSubnet"
az network vnet subnet update --name "${TEMP_PE_SUBNET_NAME}" --vnet-name $TEMP_PE_VNET_NAME --resource-group $TEMP_PE_RG_NAME --disable-private-endpoint-network-policies

# プライベート DNS ゾーンを登録するサブスクリプション ID とリソースグループ、リンク先 VNET
TEMP_PDZ_SUBSCRIPTION_ID="${SUBSCRIPTION_ID_MGMT}"
TEMP_PDZ_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
TEMP_PDZ_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-ops-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-ops-${TEMP_LOCATION_PREFIX}"

# PE作成・DNS登録（※ 設定ここまで、以降は原則的にいじらない）
for TEMP_RESOURCE_ID in $TEMP_RESOURCE_IDS; do

TEMP_RESOURCE_NAME=${TEMP_RESOURCE_ID##*/}
TEMP_GROUP_ID=$(az network private-link-resource list --id ${TEMP_RESOURCE_ID} --query "[0].properties.groupId" -o tsv)
TEMP_REQUIRED_ZONE_NAMES=$(az network private-link-resource list --id ${TEMP_RESOURCE_ID} --query "[0].properties.requiredZoneNames" -o tsv)
TEMP_PE_NAME="pe-${TEMP_RESOURCE_NAME}"

# Private Endpoint 作成
az network private-endpoint create --resource-group $TEMP_PE_RG_NAME --vnet-name $TEMP_PE_VNET_NAME --subnet "${TEMP_PE_SUBNET_NAME}" --name $TEMP_PE_NAME --private-connection-resource-id $TEMP_RESOURCE_ID --group-ids "${TEMP_GROUP_ID}"  --connection-name "${TEMP_RESOURCE_NAME}_${TEMP_PE_VNET_NAME}"

# Private DNS Zone 作成
echo "Create DNS Zones on ${TEMP_PDZ_SUBSCRIPTION_ID} ${TEMP_PDZ_RG_NAME} : ${TEMP_REQUIRED_ZONE_NAMES}"

for TEMP_REQUIRED_ZONE_NAME in $TEMP_REQUIRED_ZONE_NAMES; do
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

# Private DNS Zone Group 作成
az network private-endpoint dns-zone-group create --endpoint-name ${TEMP_PE_NAME} --name "pdzg-${TEMP_PE_NAME}" --private-dns-zone $TEMP_PRIVATE_DNS_ZONE_ID --resource-group ${TEMP_PE_RG_NAME} --zone-name "${TEMP_REQUIRED_ZONE_NAME}"

TEMP_PRIVATE_DNS_ZONE_NAMES="privatelink.blob.core.windows.net"
# Ops 側
TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
for TEMP_PRIVATE_DNS_ZONE_NAME in ${TEMP_PRIVATE_DNS_ZONE_NAMES}; do
az network private-dns zone create --resource-group ${TEMP_RG_NAME} --name ${TEMP_PRIVATE_DNS_ZONE_NAME}
# Ops VNET へリンク
TEMP_VNET_NAME="vnet-ops-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-ops-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}"
az network private-dns link vnet create --resource-group $TEMP_RG_NAME --zone-name $TEMP_PRIVATE_DNS_ZONE_NAME --name $TEMP_VNET_NAME --virtual-network $TEMP_VNET_ID --registration-enabled false
done # TEMP_PRIVATE_DNS_ZONE_NAME

done # TEMP_LOCATION

```
