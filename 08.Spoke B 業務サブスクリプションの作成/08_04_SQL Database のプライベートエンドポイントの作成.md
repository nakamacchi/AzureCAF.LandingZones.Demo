# SQL Database のプライベートエンドポイントの作成

既定の状態では、SQL Database はパブリックエンドポイントのみを持ちます。このエンドポイントを VNET 内に引き込み、プライベートエンドポイント経由で SQL Database を利用できるようにします。作業としては以下 2 つを実施します。

- プライベートエンドポイントの作成
- プライベート DNS ゾーンへの IP アドレスの登録
  - DNS ゾーングループと呼ばれる機能を利用して登録を行います。これを行うことにより、プライベートエンドポイントの利用に必要な DNS エントリをまとめてプライベート DNS ゾーンに登録することができます。

作業後、動作確認のために DNS 名を nslookup で引いてみます。

- プライベートエンドポイントを利用するためには、プライベートエンドポイントを利用したいサーバやマシンで DNS 名を引いた際に、プライベート IP アドレスに解決される必要があります。
  - まず、現在作業しているご自身の端末上で、作成した SQL Database の論理サーバ名（sql-spokeb-XXX-XXX.database.windows.net）を nslookup で引いてみてください。パブリック IP アドレスが返されるはずです。
  - 今度は Ops VNET 内の運用管理作業端末に Bastion 経由でログインし、そこで作成した SQL Database の論理サーバ名（sql-spokeb-XXX-XXX.database.windows.net）を nslookup で引いてみてください。プライベート IP アドレスが返されれば成功です。

```bash

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# Spoke B サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_B}"
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
# パラメータ
TEMP_SQL_SERVER_NAME="sql-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_SQL_DB_NAME="pubs"
TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
 
TEMP_VNET_NAME="vnet-spokeb-${TEMP_LOCATION_PREFIX}"
 
TEMP_SQL_SERVER_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourceGroups/rg-spokeb-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Sql/servers/${TEMP_SQL_SERVER_NAME}"
TEMP_PE_NAME="pe-${TEMP_SQL_SERVER_NAME}"
 
az network vnet subnet update --name "PrivateEndpointSubnet" --vnet-name $TEMP_VNET_NAME --resource-group $TEMP_RG_NAME --disable-private-endpoint-network-policies
az network private-endpoint create --name $TEMP_PE_NAME --resource-group $TEMP_RG_NAME --vnet-name $TEMP_VNET_NAME --subnet "PrivateEndpointSubnet" --private-connection-resource-id $TEMP_SQL_SERVER_ID --group-ids sqlServer --connection-name "${TEMP_SQL_SERVER_NAME}_${TEMP_VNET_NAME}"
 
# Mgmt サブスクリプションに作成済みの Private DNS Zone にレコードを登録
TEMP_PRIVATE_DNS_ZONE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-privatednszones-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/privateDnsZones/privatelink.database.windows.net"
TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
 
az network private-endpoint dns-zone-group create --endpoint-name ${TEMP_PE_NAME} --name "pdzg-${TEMP_PE_NAME}" --private-dns-zone $TEMP_PRIVATE_DNS_ZONE_ID --resource-group ${TEMP_RG_NAME} --zone-name "privatelink.database.windows.net"
 
done # TEMP_LOCATION
 
# Ops VNET の vm-ops-* から、sql-spokeb-*.database.windows.net が名前解決できることを確認

```
