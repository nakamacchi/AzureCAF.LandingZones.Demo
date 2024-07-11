# CAE・CA 用プライベート DNS 登録

CAE の外（k8s クラスタの外部）からコンテナアプリにアクセスできるように、名前解決を適切に構成します。

## CAE・CA の名前解決の仕組みについて

CAE・CA では、**プライベートエンドポイントの仕組みを利用せずに** VNET への組み込みが行われます。このため、アクセスに利用する FQDN と IP アドレスも、プライベートエンドポイントとは異なる構成になります。（プライベート DNS ゾーングループなどの機能を使って登録することができません。）

まず、CAE・CA では、以下のような形で FQDN 名と IP アドレスが決まります。

- CA を external 設定（クラスタ外へ公開する設定）で作成すると、以下のような FQDN でクラスタ外部からアクセスできるようになります。
  - https://ca-spokef-eus.niceforest-00c9b3e2.eastus.azurecontainerapps.io
- この FQDN は、以下のようになっています。
  - コンテナアプリの名前 "ca-spokef-eus" が FQDN の先頭に付きます。（※ （参考）k8s リビジョン別の FQDN も用意されます）
  - その後ろの部分（この例では .niceforest-00c9b3e2.eastus.azurecontainerapps.io）は、CAE（k8s クラスタ）に対して一意に決定される名前になっています。
- 同一の CAE 上にホストされるすべての CA は、同一の IP アドレスを持ちます。
  - 上記の例の場合、*.niceforest-00c9b3e2.eastus.azurecontainerapps.io はすべて同一の IP アドレスになります。（一つの IP アドレス上に、複数の CA が共存している形で IP アドレスが構成されます。）
  - この IP アドレスにアクセスすると、フロントリバースプロキシ（envoy）がホストヘッダーをチェックし、ホストヘッダーに対応するサービスへと HTTP リクエストを転送します。（このため、IP アドレスを使って CA にアクセスする場合には、HTTP ホストヘッダーに、アクセスしたいサービスの FQDN を指定する必要があります。）

## プライベート DNS の構成方法について

上記のような形で FQDN と IP アドレスが構成されているため、以下のようにして本番 NW からのアクセスができるように構成します。

- Hub VNET にプライベート DNS ゾーンを登録します。
  - プライベート DNS ゾーンの名前は CAE に対して決まる FQDN サフィックス（上記の例では niceforest-00c9b3e2.eastus.azurecontainerapps.io）とします。
  - この FQDN は以下の命令で取得できます。
    - az containerapp env show --name ${TEMP_CAE_NAME} --resource-group ${TEMP_RG_NAME} --query properties.defaultDomain -o tsv
- *.niceforest-00c9b3e2.eastus.azurecontainerapps.io が、すべて同一の IP アドレスに解決されるように構成します。
  - このために、A レコードとして "\*" を指定し、CAE に対して決定される一意の IP アドレスを指定します。
  - この IP アドレスは以下の命令で取得できます。
    - az containerapp env show --name ${TEMP_CAE_NAME} --resource-group ${TEMP_RG_NAME} --query properties.staticIp -o tsv

```bash

# CAE DNS 登録

# 業務システム F チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spokef_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# Spoke F サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_F}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-spokef-${TEMP_LOCATION_PREFIX}"
TEMP_CAE_NAME="cae-spokef-${TEMP_LOCATION_PREFIX}"
TEMP_CA_NAME="ca-spokef-${TEMP_LOCATION_PREFIX}"

TEMP_CAE_DEFAULT_DOMAIN=$(az containerapp env show --name ${TEMP_CAE_NAME} --resource-group ${TEMP_RG_NAME} --query properties.defaultDomain -o tsv)
TEMP_CAE_STATIC_IP=$(az containerapp env show --name ${TEMP_CAE_NAME} --resource-group ${TEMP_RG_NAME} --query properties.staticIp -o tsv)

# Hub にプライベート DNS ゾーンとして登録する
# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_F}"
 
# プライベート DNS ゾーンを登録するサブスクリプション ID とリソースグループ、リンク先 VNET
TEMP_PDZ_SUBSCRIPTION_ID="${SUBSCRIPTION_ID_HUB}"
TEMP_PDZ_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
TEMP_PDZ_VNET_ID="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourceGroups/rg-hub-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-hub-${TEMP_LOCATION_PREFIX}"

# Private DNS Zone 作成
TEMP_REQUIRED_ZONE_NAME=${TEMP_CAE_DEFAULT_DOMAIN}
echo "Create DNS Zones on ${TEMP_PDZ_SUBSCRIPTION_ID} ${TEMP_PDZ_RG_NAME} : ${TEMP_REQUIRED_ZONE_NAME}"

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

# IP アドレスを登録する
az network private-dns record-set a add-record --resource-group ${TEMP_PDZ_RG_NAME} --zone-name ${TEMP_REQUIRED_ZONE_NAME} --record-set-name "*" --ipv4-address ${TEMP_CAE_STATIC_IP} --subscription "${TEMP_PDZ_SUBSCRIPTION_ID}"

done # TEMP_LOCATION

```
