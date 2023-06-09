# SQL Database のプライベートエンドポイントの作成

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
