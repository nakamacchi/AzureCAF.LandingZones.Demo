# az cli の書き方のコツ

## ID 値がスペースを含んでいる場合

いったん JSON で取り出し、スペースをダミー文字列に変換してから再処理する。

```bash
# スペースを加味して処理するために...
# for TEMP_RESOURCE_ID in $(az resource list --query [].id -o tsv); do
TEMP_IDS=$(az resource list --query [].id)
TEMP_IDS=${TEMP_IDS//  \"/\"}
TEMP_IDS=${TEMP_IDS//[\[\],]/}
TEMP_IDS=${TEMP_IDS// /SPACEFIX}
for TEMP in $TEMP_IDS; do
  TEMP_ID="${TEMP//SPACEFIX/ }"
  TEMP_RESOURCE_ID="${TEMP_ID//\"/}"
 
 
for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_IDS; do
  az account set -s $TEMP_SUBSCRIPTION_ID
  TEMP_IDS=$(az monitor action-group list --query [].id)
  TEMP_IDS=${TEMP_IDS//  \"/\"}
  TEMP_IDS=${TEMP_IDS//[\[\],]/}
  TEMP_IDS=${TEMP_IDS// /SPACEFIX}
  # echo $TEMP_IDS
  for TEMP in $TEMP_IDS; do
    TEMP_ID="${TEMP//SPACEFIX/ }"
    TEMP_ID="${TEMP_ID//\"/}"
    echo $TEMP_ID
    az rest --method delete --uri "${TEMP_ID}?api-version=2023-01-01"
    # az monitor action-group delete --ids "$TEMP_ID"
  done
done
```

## 変数を使った検索

- 外側は "、変数は ' で囲む。
- [? ] の内側に適宜スペースを挟む。

```bash
az vm list --query "[? location == '${TEMP_LOCATION_NAME}' ].id" --subscription ${SUBSCRIPTION_ID_SPOKE_A}
```

## OS 種別判定

- osProfile の configuration で見分けるとよい
- list で検索する場合は [?osProfile.xxxConfiguration!=null] で一覧、show で一意検索する場合は以下のようにして true/false を取得

```bash

TEMP_VM_ID="/subscriptions/4104fe87-a508-4913-813c-0a23748cd402/resourceGroups/rg-spokea-eus/providers/Microsoft.Compute/virtualMachines/vm-web-eus"

TEMP_IS_WIN=$(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv)
TEMP_IS_LINUX=$(az vm show --ids "${TEMP_VM_ID}" --query osProfile.linuxConfiguration!=null -o tsv)
if [ "$TEMP_IS_LINUX" == "true" ] ; then
  echo "Linux OS です"
elif [ "$TEMP_IS_WIN" == "true" ]; then
  echo "Windows OS です"
else
  echo "不明な OS です"
fi

# または

if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then
  echo "Windows OS です"
fi

if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.linuxConfiguration!=null -o tsv) == "true" ]; then
  echo "Linux OS です"
fi

```

## az rest コマンドでインラインで JSON ファイルを記述して利用する方法

- EOF マークを利用すると直接記述ができる（ファイルをわざわざ作らなくてよい）
- EOF を 'EOF' と書くと変数展開が行われない（EOF と書くと変数展開が行われる）

```bash

az rest --method put --uri "${TEMP_COSMOSDB_ID}/sqlRoleAssignments/${TEMP_ROLE_ASSIGNMENT_ID}?api-version=2021-11-15-preview" --body @- <<EOF
{
  "properties": {
    "roleDefinitionId": "${TEMP_COSMOSDB_ID}/sqlRoleDefinitions/00000000-0000-0000-0000-000000000001",
    "scope": "${TEMP_COSMOSDB_ID}/",
    "principalId": "${TEMP_WEBAPP_MID}"
  }
}
EOF

```

## リソースオブジェクトが存在するか否かを判断した上で処理を進める方法①

- list コマンドでリソースを検索し、存在をチェックする

```bash

# KeyVault の作成 ※ KeyVault は一意名が必要なため、必要に応じて修正する
TEMP_ADE_KV_NAME="kv-spokea-ade-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
# 以前の soft-deleted リソースが残っている場合はパージしてから作成
TEMP_RES_ID=$(az keyvault list-deleted  --query "[? name =='${TEMP_ADE_KV_NAME}'].id" -o tsv)
if [[ -n $TEMP_RES_ID ]]; then
  echo "Purging soft-deleted Keyvault : " $TEMP_RES_ID
  az rest --method POST --url "${TEMP_RES_ID}/purge?api-version=2022-07-01"
  sleep 10 # Purge 完了待ち（直後に再作成すると conflict するため）
fi
az keyvault create --name $TEMP_ADE_KV_NAME --resource-group ${TEMP_RG_NAME} --location ${TEMP_LOCATION_NAME} --enabled-for-disk-encryption --bypass AzureServices --default-action Deny

```

```bash

if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='DependencyAgentWindows']" -o tsv)" ]; then
az vm identity assign --name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME}
az vm extension set --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name "AzureMonitorWindowsAgent" --publisher "Microsoft.Azure.Monitor" --enable-auto-upgrade true
az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --name "DependencyAgentWindows"  --publisher "Microsoft.Azure.Monitoring.DependencyAgent" --settings "{\"enableAMA\": \"true\"}"
# MMA は不要なので削除 (基本的にインストールされていないはずだが)
az vm extension delete --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name MicrosoftMonitoringAgent
fi

```

## リソースオブジェクトが存在するか否かを判断した上で処理を進める方法②

- rest get コマンドでリソース取得を試みて、Not Found が返された場合には処理を行うように書く。
- 2>&1 が重要。エラーメッセージは標準出力 (stdout) ではなく標準エラー出力 (stderr) に出力されるため、単に変数で受ける方法だとデータが取れない。2>&1 を記述することで、標準エラー出力を標準出力にリダイレクトし、これを変数で受けることができるようになる。
- if/then を逆に書きたい場合には、[[ ]] の中で ! を使う。（if [[ ! ${TEMP} =~ "Not Found" ]]; then とする）

```bash

echo "Checking existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
TEMP=$(az rest --uri ${TEMP_URI} --method GET 2>&1)
if [[ ${TEMP} =~ "Not Found" ]]; then
  echo "Not existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
else
  echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
  az rest --uri ${TEMP_URI} --method DELETE
fi

```

## エラーが出た場合にはエラーが出なくなるまで繰り返す方法

- 下記の例では、az monitor data-collection rule create で InvalidPayload というエラー文字列が出た場合には、しばらく待機して繰り返している
- 2>&1 が重要。エラーメッセージは標準出力 (stdout) ではなく標準エラー出力 (stderr) に出力されるため、単に変数で受ける方法だとデータが取れない。2>&1 を記述することで、標準エラー出力を標準出力にリダイレクトし、これを変数で受けることができるようになる。

```bash

TEMP="(InvalidPayload)"
while [[ ${TEMP} =~ "InvalidPayload" ]]
do
  echo "Trying to create DCR ${TEMP_DCR_FIM_NAME}..."
  sleep 10
  TEMP=$(az monitor data-collection rule create --name "${TEMP_DCR_FIM_NAME}" --resource-group "${TEMP_RG_NAME}" --location "${TEMP_LOCATION_NAME}" --rule-file dcr.json 2>&1)
done

```

## プライベートエンドポイント作成

- プライベートエンドポイント作成に必要な DNS 情報や Group ID などは、az network private-endpoint dns-zone-group list コマンドで取得できる。
- 以下のようにすることで、複数のプライベートエンドポイントをまとめて作成することが可能

``` bash

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

```

## プロビジョニング完了待機

- while ループでリソース作成の完了を待機

```bash

while [ $(az containerapp env show --resource-group "${TEMP_RG_NAME}" --name "${TEMP_CAE_NAME}" --query properties.provisioningState -o tsv) != "Succeeded" ]
do
echo "CAE provisioning State is $(az containerapp env show --resource-group "${TEMP_RG_NAME}" --name "${TEMP_CAE_NAME}" --query properties.provisioningState -o tsv) ..."
sleep 10
done

```

```bash

while true
do
  HEALTH_STATUS=$(az rest --method GET --uri "${TEMP_NC_ID}?api-version=2023-04-01" --query properties.healthCheckStatus -o tsv)
  echo "Network Connection HealthCheck State is $HEALTH_STATUS ..."
  if [ "$HEALTH_STATUS" == "Succeeded" ] || [ "$HEALTH_STATUS" == "Warning" ]; then
    break
  fi
  sleep 10
done

```


