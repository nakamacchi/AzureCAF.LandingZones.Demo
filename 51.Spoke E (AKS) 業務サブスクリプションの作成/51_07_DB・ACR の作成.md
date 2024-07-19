# DB・ACR の作成

引き続き、以下の作業を行います。

- 業務アプリで利用するデータベースと、Webアプリを含んだ Docker イメージを格納するコンテナレジストリ（ACR, Azure Container Registry）
を作成します。
- SQL DB 及び ACR に対して、本番NW、運用NWそれぞれに向けてプライベートエンドポイントを作成します。プライベートエンドポイントの DNS 設定は、それぞれ Hub VNET, Ops VNET にリンクされたプライベート DNS ゾーンに登録します。

```bash

# 業務システム E チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokee_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokee_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"

# SQL DB 作成
# （論理サーバはグローバル一意名が必要なため UNIQUE_SUFFIX を付与して名前を作る）
TEMP_SQL_SERVER_NAME="sql-spokee-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_SQL_DB_NAME="pubs"
az sql server create --name $TEMP_SQL_SERVER_NAME --resource-group $TEMP_RG_NAME --location $TEMP_LOCATION_NAME --admin-user $ADMIN_USERNAME --admin-password $ADMIN_PASSWORD --enable-public-network false
TEMP_SQLDB_OPTIONS=$( [[ "$FLAG_USE_WORKLOAD_AZ" = true ]] && echo "--compute-model Serverless --edition GeneralPurpose --family Gen5 --capacity 1 --zone-redundant true --backup-storage-redundancy Geo" || echo "--edition Basic --capacity 5" )
az sql db create --server $TEMP_SQL_SERVER_NAME --resource-group $TEMP_RG_NAME --name $TEMP_SQL_DB_NAME $TEMP_SQLDB_OPTIONS

# TLS バージョン 1.2 を最小限に設定
az sql server update --name ${TEMP_SQL_SERVER_NAME} --resource-group ${TEMP_RG_NAME} --minimal-tls-version 1.2
# (参考) 現在の設定を確認
# az sql server show --name ${TEMP_SQL_SERVER_NAME} --resource-group ${TEMP_RG_NAME} --query "minimalTlsVersion"

# Azure Container Registry 作成
# プライベートエンドポイント利用に Premium SKU が必要
TEMP_ACR_NAME="acrspokee${UNIQUE_SUFFIX}${TEMP_LOCATION_PREFIX}"
TEMP_ACR_OPTIONS=$( [[ "$FLAG_USE_WORKLOAD_AZ" = true ]] && echo "--zone-redundancy enabled" || echo "" )
az acr create --name $TEMP_ACR_NAME --resource-group $TEMP_RG_NAME --public-network-enabled false --sku Premium $TEMP_ACR_OPTIONS

done # TEMP_LOCATION

# ガバナンス管理のアカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
# スポーク E 上で作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_SQL_SERVER_NAME="sql-spokee-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_SQL_DB_NAME="pubs"
TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"

TEMP_LAW_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/law-vdc-${TEMP_LOCATION_PREFIX}"

# サーバ監査の有効化
az sql server audit-policy update --resource-group $TEMP_RG_NAME --name $TEMP_SQL_SERVER_NAME --state Enabled --lats Enabled --lawri ${TEMP_LAW_ID}

# データベース監査の有効化
az sql db audit-policy update --resource-group $TEMP_RG_NAME --server $TEMP_SQL_SERVER_NAME --name $TEMP_SQL_DB_NAME --state Enabled --lats Enabled --lawri ${TEMP_LAW_ID}

done # TEMP_LOCATION

# ======================================
# 本番NW (vnet-spokee-XXX) への PE 作成

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# PE を作成するリソースの定義
TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_SQL_SERVER_NAME="sql-spokee-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_ACR_NAME="acrspokee${UNIQUE_SUFFIX}${TEMP_LOCATION_PREFIX}"

TEMP_RESOURCE_IDS="\
/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Sql/servers/${TEMP_SQL_SERVER_NAME}
/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.ContainerRegistry/registries/${TEMP_ACR_NAME}
"

# PE を作成する VNET/Subnet
TEMP_PE_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_PE_VNET_NAME="vnet-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_PE_SUBNET_NAME="PrivateEndpointSubnet"
az network vnet subnet update --name "${TEMP_PE_SUBNET_NAME}" --vnet-name $TEMP_PE_VNET_NAME --resource-group $TEMP_PE_RG_NAME --disable-private-endpoint-network-policies

# プライベート DNS ゾーンを登録するサブスクリプション ID とリソースグループ、リンク先 VNET
TEMP_PDZ_SUBSCRIPTION_ID="${SUBSCRIPTION_ID_HUB}"
TEMP_PDZ_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
TEMP_PDZ_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourceGroups/rg-hub-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-hub-${TEMP_LOCATION_PREFIX}"

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

done # TEMP_REQUIRED_ZONE_NAME
done # TEMP_RESOURCE_ID
done # TEMP_LOCATION

# ======================================
# 運用NW (vnet-spokeemtn-XXX) への PE 作成

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_E}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# PE を作成するリソースの定義
TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_SQL_SERVER_NAME="sql-spokee-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_ACR_NAME="acrspokee${UNIQUE_SUFFIX}${TEMP_LOCATION_PREFIX}"

TEMP_RESOURCE_IDS="\
/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Sql/servers/${TEMP_SQL_SERVER_NAME}
/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.ContainerRegistry/registries/${TEMP_ACR_NAME}
"

# PE を作成する VNET/Subnet
TEMP_PE_RG_NAME="rg-spokeemtn-${TEMP_LOCATION_PREFIX}"
TEMP_PE_VNET_NAME="vnet-spokeemtn-${TEMP_LOCATION_PREFIX}"
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

done # TEMP_REQUIRED_ZONE_NAME
done # TEMP_RESOURCE_ID
done # TEMP_LOCATION

```
