# DevBox 用 Azure Firewall ルールの作成

今回の DevBox は、開発者が自由に作成・利用できる端末として用意しますが、ガバナンスの観点から、DevBox からの通信はすべて Azure Firewall を通すようにし、通信ログをすべて取得するようにします。

この際、Windows VM からは非常に多くの通信が発生するため、既知の通信については見分けられるようにしておくと便利です。このことを加味して Azure Firewall のルール（ポリシー）を作成します。

- DevBox 動作のために最低限必要な通信要件は Windows 365 と同一で、[こちら](https://learn.microsoft.com/ja-jp/windows-365/enterprise/requirements-network)です。これらに加えて、アプリケーション開発に必要な URL を適宜追加します。(GitHub, NuGet など)
- Azure Firewall を利用する場合には、[FQDNタグ](https://learn.microsoft.com/ja-jp/windows-365/enterprise/azure-firewall-windows-365)を利用できますが、ネットワークルールは手作業での設定が必要です。

```bash

# アクセス権の関係で、アカウントを一時的に切り替えて ID やエンドポイント情報を取得しておく
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

TEMP_LAW_RES_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/law-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_DCE_RES_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionEndpoints/dce-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_LAW_CUSTOMER_ID=$(az monitor log-analytics workspace show --ids ${TEMP_LAW_RES_ID} --query customerId -o tsv)
TEMP_DCE_CONFIG_EP=$(az monitor data-collection endpoint show --ids ${TEMP_DCE_RES_ID} --query configurationAccess.endpoint -o tsv)
TEMP_DCE_CONFIG_FQDN=${TEMP_DCE_CONFIG_EP:8}
TEMP_DCE_INGEST_EP=$(az monitor data-collection endpoint show --ids ${TEMP_DCE_RES_ID} --query logsIngestion.endpoint -o tsv)
TEMP_DCE_INGEST_FQDN=${TEMP_DCE_INGEST_EP:8}

# DevCenter 構築用アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_dev1_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_dev1_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# Dev1 サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_DEV1}"

# メインリージョンに開発環境を作成する
TEMP_LOCATION_NAME=${LOCATION_NAMES[0]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}

TEMP_RG_NAME="rg-devhub-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-devhub-${TEMP_LOCATION_PREFIX}"
TEMP_FW_NAME="fw-devhub-${TEMP_LOCATION_PREFIX}"
TEMP_FWP_NAME="${TEMP_FW_NAME}-fwp"

# DNS プロキシを有効化 (FQDN Network rule 有効化のため)
az network firewall policy update --name ${TEMP_FWP_NAME} --resource-group ${TEMP_RG_NAME} --enable-dns-proxy true

# https://learn.microsoft.com/en-us/rest/api/virtualnetwork/firewall-policy-rule-collection-groups/create-or-update?tabs=HTTP
az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/firewallPolicies/${TEMP_FWP_NAME}/ruleCollectionGroups/AdditionalApplicationRuleCollectionGroup?api-version=2023-05-01" --body @- <<EOF
{
  "properties": {
    "priority": 3000,
    "ruleCollections": [
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "Windows365Enterprise",
        "priority": 10000,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "Windows365Enterprise", "fqdnTags": [ "WindowsUpdate", "Windows365", "MicrosoftIntune" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Windows Push Notification",  "targetFqdns": [ "client.wns.windows.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "Additional FQDNs",  "targetFqdns": [ "login.microsoftonline.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "AllowAllTraffic",
        "priority": 50000,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Http", "port": 80 }, { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "AllowAllTraffic",  "targetFqdns": [ "*" ] }
        ]
      }
    ]
  }
}
EOF

# ポリシーの更新完了を待機
while true
do
  STATE=$(az network firewall policy show -g ${TEMP_RG_NAME} -n ${TEMP_FWP_NAME} --query provisioningState -o tsv)
  echo "Waiting for provisioning... ${STATE}"
  if [ $STATE == "Succeeded" ]; then
    break
  fi
  sleep 10
done

az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/firewallPolicies/${TEMP_FWP_NAME}/ruleCollectionGroups/AdditionalNetworkRuleCollectionGroup?api-version=2023-05-01" --body @- <<EOF
{
  "properties": {
    "priority": 4000,
    "ruleCollections": [
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "Windows365Enterprise",
        "priority": 10000,
        "rules": [
          { "ruleType": "NetworkRule", "ipProtocols": [ "TCP" ], "sourceAddresses": [ "*" ], "name": "KMS", "destinationPorts": [ 1688 ],  "destinationFqdns": [ "kms.core.windows.net", "azkms.core.windows.net" ] },
          { "ruleType": "NetworkRule", "ipProtocols": [ "TCP" ], "sourceAddresses": [ "*" ], "name": "Registration", "destinationPorts": [ 443, 5671 ],  "destinationFqdns": [ "global.azure-devices-provisioning.net", "hm-iot-in-prod-preu01.azure-devices.net", "hm-iot-in-prod-prap01.azure-devices.net", "hm-iot-in-prod-prna01.azure-devices.net", "hm-iot-in-prod-prau01.azure-devices.net", "hm-iot-in-2-prod-prna01.azure-devices.net", "hm-iot-in-3-prod-prna01.azure-devices.net", "hm-iot-in-2-prod-prna01.azure-devices.net", "hm-iot-in-3-prod-prna01.azure-devices.net" ] },
          { "ruleType": "NetworkRule", "ipProtocols": [ "UDP" ], "sourceAddresses": [ "*" ], "name": "TURN-UDP", "destinationPorts": [ 3478 ],  "destinationAddresses": [ "20.202.0.0/16", "13.107.17.41/32" ] },
          { "ruleType": "NetworkRule", "ipProtocols": [ "TCP" ], "sourceAddresses": [ "*" ], "name": "TURN-TCP", "destinationPorts": [ 443 ],  "destinationAddresses": [ "20.202.0.0/16" ] }
        ]
      }
    ]
  }
}
EOF

# ポリシーの更新完了を待機
while true
do
  STATE=$(az network firewall policy show -g ${TEMP_RG_NAME} -n ${TEMP_FWP_NAME} --query provisioningState -o tsv)
  echo "Waiting for provisioning... ${STATE}"
  if [ $STATE == "Succeeded" ]; then
    break
  fi
  sleep 10
done

```
