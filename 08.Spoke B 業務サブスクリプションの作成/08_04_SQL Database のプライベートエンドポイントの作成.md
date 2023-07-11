# SQL Database のプライベートエンドポイントの作成

既定の状態では、SQL Database はパブリックエンドポイントのみを持ちます。このエンドポイントを VNET 内に引き込み、プライベートエンドポイント経由で SQL Database を利用できるようにします。作業としては以下 2 つを実施します。

- プライベートエンドポイントの作成
- プライベート DNS ゾーンへの IP アドレスの登録
  - DNS ゾーングループと呼ばれる機能を利用して登録を行います。これを行うことにより、プライベートエンドポイントの利用に必要な DNS エントリをまとめてプライベート DNS ゾーンに登録することができます。

作業後、動作確認のために DNS 名を nslookup で引いてみます。

- プライベートエンドポイントを利用するためには、プライベートエンドポイントを利用したいサーバやマシンで DNS 名を引いた際に、プライベート IP アドレスに解決される必要があります。
  - まず、現在作業しているご自身の端末上で、作成した SQL Database の論理サーバ名（sql-spokeb-XXX-XXX.database.windows.net）を nslookup で引いてみてください。パブリック IP アドレスが返されるはずです。
  - 今度は Ops VNET 内の運用管理作業端末に Bastion 経由でログインし、そこで作成した SQL Database の論理サーバ名（sql-spokeb-XXX-XXX.database.windows.net）を nslookup で引いてみてください。プライベート IP アドレスが返されれば成功です。

- (2023/07/11 スクリプト修正) プライベートエンドポイントを 2 個作成するように修正しました。
  - プライベート DNS ゾーングループは、プライベートエンドポイント 1 つにつき 1 つしか作成できません。
  - 本サンプルの VDC 設計では、本番NW／運用NW分離要件から 2 つのネットワークを分離しています。実際には Spoke B ネットワークは両方の NW からアクセスできるようになっていますが、DNS は Hub/Ops の 2 箇所に割れているため、プライベート DNS ゾーングループは 2 つ必要になります。このため、プライベートエンドポイントを 2 つ作成し、片方は Hub 用、もう片方は Ops 用という形にしています。
  - より分離性を高めるためには、2 つのエンドポイントを作成する VNET 自体を 2 つに分ける（Spoke B NW と Spoke B 保守用 NW に分ける）形にし、それぞれにプライベートエンドポイントを降ろす設計がベストですが、今回は簡単のため、本サンプルのようにしています。

- リソース作成後、Ops VNET の vm-ops-XXX から、sql-spokeb-XXX-XXX.database.windows.net の名前解決ができることを確認してください。

```bash

TEMP_RG_NAME=""
TEMP_VNET_NAME=""
TMEP_SUBNET_NAME=""

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi

# Spoke B サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_B}"

############################################3
# ① SQL DB の prod 用プライベートエンドポイントを vnet-spokeb-XXX に作成し、Hub DNS に登録

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# PE を作成するリソースの定義
TEMP_RESOURCE_IDS="\
/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourceGroups/rg-spokeb-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Sql/servers/sql-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}
"

# PE を作成する VNET/Subnet
TEMP_PE_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_PE_VNET_NAME="vnet-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_PE_SUBNET_NAME="PrivateEndpointSubnet"
az network vnet subnet update --name "${TEMP_PE_SUBNET_NAME}" --vnet-name $TEMP_PE_VNET_NAME --resource-group $TEMP_PE_RG_NAME --disable-private-endpoint-network-policies

# プライベート DNS ゾーンを登録するサブスクリプション ID とリソースグループ
TEMP_PDZ_SUBSCRIPTION_ID="${SUBSCRIPTION_ID_HUB}"
TEMP_PDZ_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"

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
  echo "Private DNS Zone does not exist. Creating Private DNS Zone on Subscription ${TEMP_PDZ_SUBSCRIPTION_ID}."
  az network private-dns zone create --resource-group ${TEMP_PDZ_RG_NAME} --name ${TEMP_REQUIRED_ZONE_NAME} --subscription "${TEMP_PDZ_SUBSCRIPTION_ID}"
else
  echo "Private DNS Zone already exists."
fi

# Private DNS Zone Group 作成
az network private-endpoint dns-zone-group create --endpoint-name ${TEMP_PE_NAME} --name "pdzg-${TEMP_PE_NAME}" --private-dns-zone $TEMP_PRIVATE_DNS_ZONE_ID --resource-group ${TEMP_PE_RG_NAME} --zone-name "${TEMP_REQUIRED_ZONE_NAME}"

done # TEMP_REQUIRED_ZONE_NAME
done # TEMP_RESOURCE_ID
done # TEMP_LOCATION

############################################3
# ② SQL DB の ops 用プライベートエンドポイントを vnet-spokeb-XXX に作成し、Ops DNS に登録

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# PE を作成するリソースの定義
TEMP_RESOURCE_IDS="\
/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourceGroups/rg-spokeb-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Sql/servers/sql-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}
"

# PE を作成する VNET/Subnet
TEMP_PE_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_PE_VNET_NAME="vnet-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_PE_SUBNET_NAME="PrivateEndpointSubnet"
az network vnet subnet update --name "${TEMP_PE_SUBNET_NAME}" --vnet-name $TEMP_PE_VNET_NAME --resource-group $TEMP_PE_RG_NAME --disable-private-endpoint-network-policies

# プライベート DNS ゾーンを登録するサブスクリプション ID とリソースグループ
TEMP_PDZ_SUBSCRIPTION_ID="${SUBSCRIPTION_ID_MGMT}"
TEMP_PDZ_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"

# PE作成・DNS登録（※ 設定ここまで、以降は原則的にいじらないが、今回は同一 Subnet に PE を 2 つ作成するので TEMP_PE_NAME を微修正）
for TEMP_RESOURCE_ID in $TEMP_RESOURCE_IDS; do

TEMP_RESOURCE_NAME=${TEMP_RESOURCE_ID##*/}
TEMP_GROUP_ID=$(az network private-link-resource list --id ${TEMP_RESOURCE_ID} --query "[0].properties.groupId" -o tsv)
TEMP_REQUIRED_ZONE_NAMES=$(az network private-link-resource list --id ${TEMP_RESOURCE_ID} --query "[0].properties.requiredZoneNames" -o tsv)
TEMP_PE_NAME="pe2-${TEMP_RESOURCE_NAME}"

# Private Endpoint 作成
az network private-endpoint create --resource-group $TEMP_PE_RG_NAME --vnet-name $TEMP_PE_VNET_NAME --subnet "${TEMP_PE_SUBNET_NAME}" --name $TEMP_PE_NAME --private-connection-resource-id $TEMP_RESOURCE_ID --group-ids "${TEMP_GROUP_ID}"  --connection-name "${TEMP_RESOURCE_NAME}_${TEMP_PE_VNET_NAME}"

# Private DNS Zone 作成
echo "Create DNS Zones on ${TEMP_PDZ_SUBSCRIPTION_ID} ${TEMP_PDZ_RG_NAME} : ${TEMP_REQUIRED_ZONE_NAMES}"

for TEMP_REQUIRED_ZONE_NAME in $TEMP_REQUIRED_ZONE_NAMES; do
TEMP_PRIVATE_DNS_ZONE_ID="/subscriptions/${TEMP_PDZ_SUBSCRIPTION_ID}/resourceGroups/${TEMP_PDZ_RG_NAME}/providers/Microsoft.Network/privateDnsZones/${TEMP_REQUIRED_ZONE_NAME}"

TEMP_ISEXISTS=$(az rest --method get --uri "${TEMP_PRIVATE_DNS_ZONE_ID}?api-version=2020-06-01" --query id -o tsv)
if [[ $TEMP_ISEXISTS == *"ERROR"* || -z $TEMP_ISEXISTS ]]; then
  echo "Private DNS Zone does not exist. Creating Private DNS Zone on Subscription ${TEMP_PDZ_SUBSCRIPTION_ID}."
  az network private-dns zone create --resource-group ${TEMP_PDZ_RG_NAME} --name ${TEMP_REQUIRED_ZONE_NAME} --subscription "${TEMP_PDZ_SUBSCRIPTION_ID}"
else
  echo "Private DNS Zone already exists."
fi

# Private DNS Zone Group 作成
az network private-endpoint dns-zone-group create --endpoint-name ${TEMP_PE_NAME} --name "pdzg-${TEMP_PE_NAME}" --private-dns-zone $TEMP_PRIVATE_DNS_ZONE_ID --resource-group ${TEMP_PE_RG_NAME} --zone-name "${TEMP_REQUIRED_ZONE_NAME}"

done # TEMP_REQUIRED_ZONE_NAME
done # TEMP_RESOURCE_ID
done # TEMP_LOCATION

echo "nslookup sql-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}.database.windows.net"

```
