# DevCenter へのネットワーク接続のアタッチ

DevBox を収容するネットワーク（サブネット）に対して、ネットワーク接続オブジェクト（Network Connection）を作成します。

- ネットワーク接続オブジェクトを作成すると、当該サブネット上に NIC が 1 つ作成されます。この NIC は、当該ネットワーク（サブネット）が DevBox 開発マシンを収容する上での前提条件（通信要件）を満たしているか否かをチェックする目的で利用されます。NIC は、ネットワーク接続オブジェクト作成時に指定したリソースグループに作成されます（新規のリソースグループが必要）。
- ネットワーク接続オブジェクトが作成されたら、これを DevCenter にアタッチし、DevCenter から利用できるようにします。
- 作成後、ヘルスチェックを走らせて、ネットワーク要件が満たされているかを確認します。

注意事項として、

- Azure ポータル上は、「ネットワーク接続（Network Connection）」と「アタッチされたネットワーク（Attached Network）」が表記上区別されていません（いずれも「ネットワーク接続」とのみ書かれています）。単独で存在しているのがネットワーク接続オブジェクト、これを DevCenter に組み込んだものがアタッチドネットワークオブジェクトです。
- ネットワーク接続を作成する際、NIC を作成するためのリソースグループの指定が必要になります。この際、複数のネットワーク接続でリソースグループを共用することができません。ネットワーク接続単位に異なるリソースグループ名を指定してください。

![picture 9](./images/329073a138ffd0a2e68ae23dba44f7d8a2575629c11ab0407bc787da3f8acf28.png)  

```bash

# DevCenter 構築用アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_dev1_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_dev1_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# Dev1 サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_DEV1}"

TEMP_LOCATION_NAME=${LOCATION_NAMES[0]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}

TEMP_NC_DEFS="\
/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devbox-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-devbox-${TEMP_LOCATION_PREFIX}/subnets/DevProjectXSubnet,/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devbox-${TEMP_LOCATION_PREFIX}/providers/Microsoft.DevCenter/networkConnections/nc-vnet-devbox-${TEMP_LOCATION_PREFIX}-subnet-devprojectx,/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devcenter-${TEMP_LOCATION_PREFIX}/providers/Microsoft.DevCenter/devcenters/dc-devcenter-${TEMP_LOCATION_PREFIX}/attachednetworks/an-vnet-devbox-${TEMP_LOCATION_PREFIX}-subnet-devprojectx,MNIC_rg-devbox-${TEMP_LOCATION_PREFIX}-devprojectx \
/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devbox-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-devbox-${TEMP_LOCATION_PREFIX}/subnets/DevProjectYSubnet,/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devbox-${TEMP_LOCATION_PREFIX}/providers/Microsoft.DevCenter/networkConnections/nc-vnet-devbox-${TEMP_LOCATION_PREFIX}-subnet-devprojecty,/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devcenter-${TEMP_LOCATION_PREFIX}/providers/Microsoft.DevCenter/devcenters/dc-devcenter-${TEMP_LOCATION_PREFIX}/attachednetworks/an-vnet-devbox-${TEMP_LOCATION_PREFIX}-subnet-devprojecty,MNIC_rg-devbox-${TEMP_LOCATION_PREFIX}-devprojecty \
"

for TEMP_NC_DEF in $TEMP_NC_DEFS; do
# 分解して利用
TEMP=(${TEMP_NC_DEF//,/ })

TEMP_SUBNET_ID=${TEMP[0]} # "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devbox-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Network/virtualNetworks/vnet-devbox-${TEMP_LOCATION_PREFIX}/subnets/DevBoxPool1Subnet"
TEMP_NC_ID=${TEMP[1]} # "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devbox-${TEMP_LOCATION_PREFIX}/providers/Microsoft.DevCenter/networkConnections/nc-devbox-pool1-${TEMP_LOCATION_PREFIX}"
TEMP_AN_ID=${TEMP[2]} # "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/rg-devcenter-${TEMP_LOCATION_PREFIX}/providers/Microsoft.DevCenter/devcenters/dc-devcenter-${TEMP_LOCATION_PREFIX}/attachednetworks/nc-devbox-pool1-${TEMP_LOCATION_PREFIX}"
TEMP_NC_RG_NAME=${TEMP[3]} # "rg-devbox-${TEMP_LOCATION_PREFIX}-NIC-pool1"

echo "Creating Network Connection ${TEMP_NC_ID} on ${TEMP_SUBNET_ID}"

# 1. ネットワーク接続の作成
# これにより $TEMP_SUBNET_ID に Windows 365 が管理に利用する Managed NIC が作成される（networkingResourceGroupName で指定したリソースグループに作成される、新規リソースグループ名を指定する）

az rest --method PUT --uri "${TEMP_NC_ID}?api-version=2023-04-01" --body @- <<EOF
{
  "location": "${TEMP_LOCATION_NAME}",
  "properties": {
    "domainJoinType": "AzureADJoin",
    "networkingResourceGroupName": "${TEMP_NC_RG_NAME}",
    "subnetId": "${TEMP_SUBNET_ID}"
  }
}
EOF

# az cli の場合は以下
# az devcenter admin network-connection create --domain-join-type AzureADJoin --name $TEMP_NC_NAME --resource-group $TEMP_RG_NAME --subnet-id $TEMP_SUBNET_ID

# ネットワーク接続の作成完了待ち
while true
do
  STATUS=$(az rest --method GET --uri "${TEMP_NC_ID}?api-version=2023-04-01" --query properties.provisioningState -o tsv)
  echo "Network Connection provisioning State is $STATUS ..."
  if [ "$STATUS" == "Succeeded" ]; then
    break
  fi
  sleep 10
done

# ヘルスチェックを走らせる
az rest --method POST --uri "${TEMP_NC_ID}/runHealthChecks?api-version=2023-04-01"

# 2. アタッチネットワークを作成
# DevCenter へ 1 をアタッチ
# （Network Connection と Attached Network で名前をずらす必要は特にないため、同一名を利用）
az rest --method PUT --uri "${TEMP_AN_ID}?api-version=2023-04-01" --body @- <<EOF
{
  "properties": {
    "networkConnectionId": "${TEMP_NC_ID}"
  }
}
EOF

# az cli の場合は以下
# az devcenter admin attached-network create --attached-network-connection-name ${TEMP_ANC_NAME} --dev-center ${TEMP_DC_NAME} --resource-group $TEMP_RG_NAME --network-connection-id ${TEMP_NC_ID}

done # TEMP_NC_DEF

# ヘルスチェック完了待ち
# ※ ヘルスチェックが完了しなくてもアタッチは可能なため、先に接続作成とアタッチだけを一通り行ってからヘルスチェックを実施

for TEMP_NC_DEF in $TEMP_NC_DEFS; do
# 分解して利用
TEMP=(${TEMP_NC_DEFS//,/ })
TEMP_SUBNET_ID=${TEMP[0]}
TEMP_NC_ID=${TEMP[1]}
TEMP_AN_ID=${TEMP[2]}
TEMP_NC_RG_NAME=${TEMP[3]}

while true
do
  HEALTH_STATUS=$(az rest --method GET --uri "${TEMP_NC_ID}?api-version=2023-04-01" --query properties.healthCheckStatus -o tsv)
  echo "Network Connection HealthCheck State is $HEALTH_STATUS ..."
  if [ "$HEALTH_STATUS" == "Passed" ] || [ "$HEALTH_STATUS" == "Warning" ]; then
    break
  fi
  sleep 10
done

done # TEMP_NC_DEF

```
