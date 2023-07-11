# Web App の閉域化

Web App は SQL Database と同様、既定ではパブリックエンドポイントを使って構成されています。このため、仮想ネットワークに閉じ込めて利用するための作業を行います。SQL Database の場合と異なり、入口と出口の両方について VNET への組み込みが必要になります。

- 出力ロックダウン (Egress Lockdown)
  - Web アプリから SQL DB へのアクセスのように、Web アプリから「出ていく」通信を VNET の中に閉じるようにするため、リージョナル VNET 統合機能（単に VNET 統合機能とも呼ばれます）を利用します。
  - 既定では一部の通信しか VNET 内にルーティングされず、またパブリック DNS を引くためにプライベートエンドポイント名を解決できません。このため、Web App に以下 2 つの設定を行います。
    - WEBSITE_VNET_ROUTE_ALL = 1
    - WEBSITE_DNS_SERVER = Azure Firewall の IP アドレス
- 入力ロックダウン（Ingress Lockdown）
  - Web アプリにアクセスするための経路として、パブリックエンドポイントではなくプライベートエンドポイントを利用するように構成します。（プライベートエンドポイントの作成とプライベート DNS ゾーンを構成します。）
  - Web App のアプリの管理には、Kudu と呼ばれる Web ベースの管理ツールを利用します。この Web ベース管理ツールへのアクセスにも、プライベートエンドポイントを利用するように構成する必要があります。
    - Web App の FQDN が https://XXXXX.azurewebsites.net/ の場合、Kudu へアクセスするための FQDN は https://XXXXX.scm.azurewebsites.net/ になります。
    - DNS ゾーンリンク機能を利用すると、この Kudu の FQDN もまとめてプライベート DNS ゾーンに登録されます。

作業後、動作確認のために DNS 名を nslookup で引いてみます。

- まず、現在作業しているご自身の端末上で、作成した Web App のサーバ名（webapp-spokeb-XXX-XXX.azurewebsites.net）を nslookup で引いてみてください。パブリック IP アドレスが返されるはずです。
- 今度は Ops VNET 内の運用管理作業端末に Bastion 経由でログインし、そこで作成した Web App のサーバ名（webapp-spokeb-XXX-XXX.azurewebsites.net）を nslookup で引いてみてください。プライベート IP アドレスが返されれば成功です。

```bash

# Web App の閉域化
# ネットワークに関わる作業のため、NW チームに依頼する
# ただし、一部の作業については業務システム B の開発側でもできてしまう。
# 例：強制ルーティングと DNS サーバの指定
# https://docs.microsoft.com/ja-jp/azure/app-service/web-sites-integrate-with-vnet#regional-vnet-integration
# これらの設定は、業務側が Network Contributor の権限を持っていなくても変更できてしまう
# 逆に、NW 管理チーム側の Network Contributor 権限だけでは変更できない
# ネットワークに関わる作業の一部として、NW 管理チーム側で Microsoft.Web/sites/config/write などの権限も保有する必要がある。

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi

# Spoke B サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_B}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_ASP_WIN_NAME="aspw-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_WEBAPP_NAME="webapp-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-spokeb-${TEMP_LOCATION_PREFIX}"

# 出力ロックダウン（Egress Lockdown）==================================
# Regional VNET Integration (VNET 統合 v2) の作成
# https://docs.microsoft.com/ja-jp/azure/app-service/web-sites-integrate-with-vnet#regional-vnet-integration
az webapp vnet-integration add --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --subnet "AppServiceBackendSubnet" --vnet ${TEMP_VNET_NAME}

az account set -s "${SUBSCRIPTION_NAME_HUB}"
TEMP_FW_IP=$(az network firewall ip-config list -g "rg-hub-${TEMP_LOCATION_PREFIX}" -f "fw-hub-${TEMP_LOCATION_PREFIX}" --query "[0].privateIpAddress" --output tsv)
az account set -s "${SUBSCRIPTION_NAME_SPOKE_B}"

az webapp config appsettings set --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --settings "WEBSITE_VNET_ROUTE_ALL=1"
az webapp config appsettings set --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --settings "WEBSITE_DNS_SERVER=${TEMP_FW_IP}"

done # TEMP_LOCATION

# 入力ロックダウン（Ingress Lockdown） ===============================

# Spoke B サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_B}"

############################################3
# ① Web App の prod 用プライベートエンドポイントを vnet-spokeb-XXX に作成し、Hub DNS に登録

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# PE を作成するリソースの定義
TEMP_RESOURCE_IDS="\
/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourceGroups/rg-spokeb-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Web/sites/webapp-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}
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
# ② Web App の ops 用プライベートエンドポイントを vnet-spokeb-XXX に作成し、Ops DNS に登録

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# PE を作成するリソースの定義
TEMP_RESOURCE_IDS="\
/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourceGroups/rg-spokeb-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Web/sites/webapp-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}
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


#####################################################################
# ワーカープロセスのリセット
# ネットワーク関連の設定を変えたらアプリケーションをリセット（アプリケーションリスタートではなくプロセスリセットをしないと反映されないため）
az webapp restart --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME}

done # TEMP_LOCATION

# Ops VNET の vm-ops-* から、webapp-spokeb-*.azurewebsites.net が名前解決できることを確認
echo "webapp-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}.azurewebsites.net"

```
