# Private DNS Zones の作成

```bash
# 共通基盤管理チーム／① 初期構築時の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_plat_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# 管理サブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_MGMT}"
 
# DNS ゾーンは VDC 全体で共有されるため、Mgmt サブスクリプションの専用リソースグループに作成。
# Private Endpoint は Hub VNET, Ops VNET どちらからも利用されるため、それぞれにリンクする
# プライベートリンクに関わるゾーンとしては以下のようなものがある
# https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns
# TEMP_PRIVATE_DNS_ZONE_NAMES="privatelink.adf.azure.com privatelink.afs.azure.net privatelink.agentsvc.azure-automation.net privatelink.analysis.windows.net privatelink.api.azureml.ms privatelink.azconfig.io privatelink.azure-api.net privatelink.azure-automation.net privatelink.azurecr.io privatelink.azure-devices.net privatelink.azure-devices-provisioning.net privatelink.azurehdinsight.net privatelink.azurehealthcareapis.com privatelink.azurestaticapps.net privatelink.azuresynapse.net privatelink.azurewebsites.net privatelink.batch.azure.com privatelink.blob.core.windows.net privatelink.cassandra.cosmos.azure.com privatelink.cognitiveservices.azure.com privatelink.database.windows.net privatelink.datafactory.azure.net privatelink.dev.azuresynapse.net privatelink.dfs.core.windows.net privatelink.dicom.azurehealthcareapis.com privatelink.digitaltwins.azure.net privatelink.directline.botframework.com privatelink.documents.azure.com privatelink.eventgrid.azure.net privatelink.file.core.windows.net privatelink.gremlin.cosmos.azure.com privatelink.guestconfiguration.azure.com privatelink.his.arc.azure.com privatelink.kubernetesconfiguration.azure.com privatelink.managedhsm.azure.net privatelink.mariadb.database.azure.com privatelink.media.azure.net privatelink.mongo.cosmos.azure.com privatelink.monitor.azure.com privatelink.mysql.database.azure.com privatelink.notebooks.azure.net privatelink.ods.opinsights.azure.com privatelink.oms.opinsights.azure.com privatelink.pbidedicated.windows.net privatelink.postgres.database.azure.com privatelink.prod.migration.windowsazure.com privatelink.purview.azure.com privatelink.purviewstudio.azure.com privatelink.queue.core.windows.net privatelink.redis.cache.windows.net privatelink.redisenterprise.cache.azure.net privatelink.search.windows.net privatelink.service.signalr.net privatelink.servicebus.windows.net privatelink.siterecovery.windowsazure.com privatelink.sql.azuresynapse.net privatelink.table.core.windows.net privatelink.table.cosmos.azure.com privatelink.tip1.powerquery.microsoft.com privatelink.token.botframework.com privatelink.vaultcore.azure.net privatelink.web.core.windows.net privatelink.webpubsub.azure.com"
 
# 上記をすべて登録するのではなく、必要なものだけ登録する
# ※ Private Link を構成していないものまで登録すると正しく動作しなくなることがある
# 例）privatelink.guestconfiguration.azure.com は PLS を構成していないと Azure Provided DNS (168.63.129.16) が agentserviceapi.guestconfiguration.azure.com を正しく引けなくなる
TEMP_PRIVATE_DNS_ZONE_NAMES="privatelink.azurewebsites.net privatelink.database.windows.net"
 
# リソースグループ作成、プライベート DNS ゾーン作成
# DC 全体で共有されるため、メインリージョン側のみに作成
TEMP_MAIN_LOCATION=${LOCATION_NAMES[0]}
TEMP_MAIN_RG_NAME="rg-privatednszones-${LOCATION_PREFIXS[0]}"
az group create --name ${TEMP_MAIN_RG_NAME} --location ${TEMP_LOCATION_NAME}
 
for TEMP_PRIVATE_DNS_ZONE_NAME in ${TEMP_PRIVATE_DNS_ZONE_NAMES}; do
  az network private-dns zone create --resource-group ${TEMP_MAIN_RG_NAME} --name ${TEMP_PRIVATE_DNS_ZONE_NAME}
done
 
# Hub, Ops VNET へリンク
for TEMP_PRIVATE_DNS_ZONE_NAME in ${TEMP_PRIVATE_DNS_ZONE_NAMES}; do
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
# Hub VNET へリンク
TEMP_VNET_NAME="vnet-hub-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourceGroups/rg-hub-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}"
az network private-dns link vnet create --resource-group $TEMP_MAIN_RG_NAME --zone-name $TEMP_PRIVATE_DNS_ZONE_NAME --name $TEMP_VNET_NAME --virtual-network $TEMP_VNET_ID --registration-enabled false
 
# Ops VNET へリンク
TEMP_VNET_NAME="vnet-ops-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-ops-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}"
az network private-dns link vnet create --resource-group $TEMP_MAIN_RG_NAME --zone-name $TEMP_PRIVATE_DNS_ZONE_NAME --name $TEMP_VNET_NAME --virtual-network $TEMP_VNET_ID --registration-enabled false
 
done # TEMP_LOCATION
done # TEMP_PRIVATE_DNS_ZONE_NAME

```

