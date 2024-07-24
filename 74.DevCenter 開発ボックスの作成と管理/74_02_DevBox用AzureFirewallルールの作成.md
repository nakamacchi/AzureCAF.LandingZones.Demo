# DevBox 用 Azure Firewall ルールの作成

今回の DevBox は、開発者が自由に作成・利用できる端末として用意しますが、ガバナンスの観点から、DevBox からの通信はすべて Azure Firewall を通すようにし、通信ログをすべて取得するようにします。

この際、Windows VM からは非常に多くの通信が発生するため、既知の通信については見分けられるようにしておくと便利です。このことを加味して Azure Firewall のルール（ポリシー）を作成します。

- DevBox 動作のために最低限必要な通信要件は Windows 365 と同一で、[こちら](https://learn.microsoft.com/ja-jp/windows-365/enterprise/requirements-network)です。これらに加えて、アプリケーション開発に必要な URL を適宜追加します。(GitHub, NuGet など)
- Azure Firewall を利用する場合には、[FQDNタグ](https://learn.microsoft.com/ja-jp/windows-365/enterprise/azure-firewall-windows-365)を利用できますが、ネットワークルールは手作業での設定が必要です。

作成する Azure Firewall のポリシーは以下のようになっています。

- DefaultApplicationRuleCollectionGroup (1000)
  - 10000 : Windows 365, Windows Update, Microsoft Intune などの基本的な通信を許可
  - 20000 : MDfS (MDE, Qualys, AMA, GC など) に関連する通信を許可
  - 30000 : OS のアップデートに関する通信を許可
  - 40000 : ユーザ用
  - 50000 : すべてのトラフィックを許可
- DefaultNetworkRuleCollectionGroup (2000)
  - 10000 : Windows 365 の基本的な通信を許可
  - 30000 : OS の時刻同期に関する通信を許可

```bash

# アクセス権の関係で、アカウントを一時的に切り替えて ID やエンドポイント情報を取得しておく
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

TEMP_LAW_RES_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/law-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_DCE_RES_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionEndpoints/dce-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_LAW_CUSTOMER_ID=$(az monitor log-analytics workspace show --ids ${TEMP_LAW_RES_ID} --query customerId -o tsv)
TEMP_DCE_CONFIG_EP=$(az monitor data-collection endpoint show --ids ${TEMP_DCE_RES_ID} --query configurationAccess.endpoint -o tsv)
TEMP_DCE_CONFIG_FQDN=${TEMP_DCE_CONFIG_EP:8}
TEMP_DCE_INGEST_EP=$(az monitor data-collection endpoint show --ids ${TEMP_DCE_RES_ID} --query logsIngestion.endpoint -o tsv)
TEMP_DCE_INGEST_FQDN=${TEMP_DCE_INGEST_EP:8}

# DevCenter 構築用アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_dev1_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi

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
az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/firewallPolicies/${TEMP_FWP_NAME}/ruleCollectionGroups/DefaultApplicationRuleCollectionGroup?api-version=2023-05-01" --body @- <<EOF
{
  "properties": {
    "priority": 1000,
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
        "name": "MDfS(MDE)",
        "priority": 20000,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "CRL",  "targetFqdns": [ "crl.microsoft.com", "ctldl.windowsupdate.com", "www.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Common",  "targetFqdns": [ "events.data.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Common (Mac/Linux)",  "targetFqdns": [ "x.cp.wd.microsoft.com", "cdn.x.cp.wd.microsoft.com", "officecdn-microsoft-com.akamaized.net" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Common (Linux)",  "targetFqdns": [ "packages.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Microsoft Defender for Endpoint US",  "targetFqdns": [ "unitedstates.x.cp.wd.microsoft.com", "us-v20.events.data.microsoft.com", "winatp-gw-cus.microsoft.com", "winatp-gw-eus.microsoft.com", "winatp-gw-cus3.microsoft.com", "winatp-gw-eus3.microsoft.com", "automatedirstrprdcus.blob.core.windows.net", "automatedirstrprdeus.blob.core.windows.net", "automatedirstrprdcus3.blob.core.windows.net", "automatedirstrprdeus3.blob.core.windows.net", "ussus1eastprod.blob.core.windows.net", "ussus2eastprod.blob.core.windows.net", "ussus3eastprod.blob.core.windows.net", "ussus4eastprod.blob.core.windows.net", "wsus1eastprod.blob.core.windows.net", "wsus2eastprod.blob.core.windows.net", "ussus1westprod.blob.core.windows.net", "ussus2westprod.blob.core.windows.net", "ussus3westprod.blob.core.windows.net", "ussus4westprod.blob.core.windows.net", "wsus1westprod.blob.core.windows.net", "wsus2westprod.blob.core.windows.net" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Microsoft Defender for Endpoint EU",  "targetFqdns": [ "europe.x.cp.wd.microsoft.com", "eu-v20.events.data.microsoft.com", "winatp-gw-neu.microsoft.com", "winatp-gw-weu.microsoft.com", "winatp-gw-neu3.microsoft.com", "winatp-gw-weu3.microsoft.com", "automatedirstrprdneu.blob.core.windows.net", "automatedirstrprdweu.blob.core.windows.net", "automatedirstrprdneu3.blob.core.windows.net", "automatedirstrprdweu3.blob.core.windows.net", "usseu1northprod.blob.core.windows.net", "wseu1northprod.blob.core.windows.net", "usseu1westprod.blob.core.windows.net", "wseu1westprod.blob.core.windows.net" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Microsoft Defender for Endpoint UK",  "targetFqdns": [ "unitedkingdom.x.cp.wd.microsoft.com", "uk-v20.events.data.microsoft.com", "winatp-gw-uks.microsoft.com", "winatp-gw-ukw.microsoft.com", "automatedirstrprduks.blob.core.windows.net", "automatedirstrprdukw.blob.core.windows.net", "ussuk1southprod.blob.core.windows.net", "wsuk1southprod.blob.core.windows.net", "ussuk1westprod.blob.core.windows.net", "wsuk1westprod.blob.core.windows.net" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Required when using Security Management for Microsoft Defender for Endpoint",  "targetFqdns": [ "enterpriseregistration.windows.net", "*.dm.microsoft.com", "login.microsoftonline.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "MDfS(DefenderAntivirus)",
        "priority": 20100,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "MU / WU",  "targetFqdns": [ "go.microsoft.com", "definitionupdates.microsoft.com", "www.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Symbols",  "targetFqdns": [ "msdl.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "MAPS",  "targetFqdns": [ "*.wdcp.microsoft.com", "*.wd.microsoft.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "MDfS(Qualys)",
        "priority": 20200,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "QualysCloudScanner",  "targetFqdns": [ "qagpublic.qg3.apps.qualys.com", "qagpublic.qg2.apps.qualys.eu" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "Azure Monitor Agent",
        "priority": 20300,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Http", "port": 80 }, { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Azure Monitor Agent", "targetFqdns": [ "global.handler.control.monitor.azure.com", "${TEMP_LOCATION_NAME}.handler.control.monitor.azure.com", "management.azure.com", "${TEMP_LOCATION_NAME}.monitoring.azure.com", "${TEMP_LAW_CUSTOMER_ID}.ods.opinsights.azure.com", "${TEMP_DCE_CONFIG_FQDN}", "${TEMP_DCE_INGEST_FQDN}" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "Guest Configuration Agent",
        "priority": 20400,
        "rules": [ 
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Guest Configuration Agent", "targetFqdns": [ "agentserviceapi.guestconfiguration.azure.com", "*.guestconfiguration.azure.com", "oaasguestconfigac2s1.blob.core.windows.net", "oaasguestconfigacs1.blob.core.windows.net", "oaasguestconfigaes1.blob.core.windows.net", "oaasguestconfigases1.blob.core.windows.net", "oaasguestconfigbrses1.blob.core.windows.net", "oaasguestconfigbrss1.blob.core.windows.net", "oaasguestconfigccs1.blob.core.windows.net", "oaasguestconfigces1.blob.core.windows.net", "oaasguestconfigcids1.blob.core.windows.net", "oaasguestconfigcuss1.blob.core.windows.net", "oaasguestconfigeaps1.blob.core.windows.net", "oaasguestconfigeas1.blob.core.windows.net", "oaasguestconfigeus2s1.blob.core.windows.net", "oaasguestconfigeuss1.blob.core.windows.net", "oaasguestconfigfcs1.blob.core.windows.net", "oaasguestconfigfss1.blob.core.windows.net", "oaasguestconfiggewcs1.blob.core.windows.net", "oaasguestconfiggns1.blob.core.windows.net", "oaasguestconfiggwcs1.blob.core.windows.net", "oaasguestconfigjiws1.blob.core.windows.net", "oaasguestconfigjpes1.blob.core.windows.net", "oaasguestconfigjpws1.blob.core.windows.net", "oaasguestconfigkcs1.blob.core.windows.net", "oaasguestconfigkss1.blob.core.windows.net", "oaasguestconfigncuss1.blob.core.windows.net", "oaasguestconfignes1.blob.core.windows.net", "oaasguestconfignres1.blob.core.windows.net", "oaasguestconfignrws1.blob.core.windows.net", "oaasguestconfigqacs1.blob.core.windows.net", "oaasguestconfigsans1.blob.core.windows.net", "oaasguestconfigscuss1.blob.core.windows.net", "oaasguestconfigseas1.blob.core.windows.net", "oaasguestconfigsecs1.blob.core.windows.net", "oaasguestconfigsfns1.blob.core.windows.net", "oaasguestconfigsfws1.blob.core.windows.net", "oaasguestconfigsids1.blob.core.windows.net", "oaasguestconfigstzns1.blob.core.windows.net", "oaasguestconfigswcs1.blob.core.windows.net", "oaasguestconfigswns1.blob.core.windows.net", "oaasguestconfigswss1.blob.core.windows.net", "oaasguestconfigswws1.blob.core.windows.net", "oaasguestconfiguaecs1.blob.core.windows.net", "oaasguestconfiguaens1.blob.core.windows.net", "oaasguestconfigukss1.blob.core.windows.net", "oaasguestconfigukws1.blob.core.windows.net", "oaasguestconfigwcuss1.blob.core.windows.net", "oaasguestconfigwes1.blob.core.windows.net", "oaasguestconfigwids1.blob.core.windows.net", "oaasguestconfigwus2s1.blob.core.windows.net", "oaasguestconfigwus3s1.blob.core.windows.net", "oaasguestconfigwuss1.blob.core.windows.net" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "WindowsOS",
        "priority": 30000,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "WindowsUpdate", "fqdnTags": [ "WindowsUpdate", "WindowsDiagnostics", "MicrosoftActiveProtectionService" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "CRL",  "targetFqdns": [ "rb.symcd.com", "s.symcd.com", "ts-ocsp.ws.symantec.com", "ts-crl.ws.symantec.com", "crl.microsoft.com", "ocsp.digicert.com", "crl4.digicert.com", "crl3.digicert.com", "mscrl.microsoft.com", "*.verisign.com", "*.entrust.net", "*.crl3.digicert.com", "*.crl4.digicert.com", "*.ocsp.digicert.com", "*.www.d-trust.net", "*.root-c3-ca2-2009.ocsp.d-trust.net", "*.crl.microsoft.com", "*.oneocsp.microsoft.com", "*.ocsp.msocsp.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "UserExperienceAndDiagnosticsComponents",  "targetFqdns": [ "v10c.events.data.microsoft.com", "v10.events.data.microsoft.com", "v10.vortex-win.data.microsoft.com", "vortex-win.data.microsoft.com", "v20.events.data.microsoft.com", "settings-win.data.microsoft.com", "watson.telemetry.microsoft.com", "umwatsonc.events.data.microsoft.com", "*-umwatsonc.events.data.microsoft.com", "ceuswatcab01.blob.core.windows.net", "ceuswatcab02.blob.core.windows.net", "eaus2watcab01.blob.core.windows.net", "eaus2watcab02.blob.core.windows.net", "weus2watcab01.blob.core.windows.net", "weus2watcab02.blob.core.windows.net", "login.live.com", "oca.microsoft.com", "kmwatsonc.telemetry.microsoft.com", "*-kmwatsonc.telemetry.microsoft.com", "settings-win.data.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "InternetConnectTest",  "targetFqdns": [ "www.msftconnecttest.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "SmartScreen",  "targetFqdns": [ "*.smartscreen.microsoft.com", "urs.microsoft.com", "unitedstates.smartscreen-prod.microsoft.com", "*.urs.microsoft.com", "checkappexec.microsoft.com", "wdcpalt.microsoft.com", "smartscreen-prod.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "LicenseActivation",  "targetFqdns": [ "activation.sls.microsoft.com", "crl.microsoft.com", "validation.sls.microsoft.com", "activation-v2.sls.microsoft.com", "validation-v2.sls.microsoft.com", "displaycatalog.mp.microsoft.com", "licensing.mp.microsoft.com", "purchase.mp.microsoft.com", "displaycatalog.md.mp.microsoft.com", "licensing.md.mp.microsoft.com", "purchase.md.mp.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "MicrosoftEdge",  "targetFqdns": [ "config.edge.skype.com", "msedge.api.cdp.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Windows Push Notification",  "targetFqdns": [ "client.wns.windows.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "UbuntuOS",
        "priority": 30100,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "UpdatePackage",  "targetFqdns": [ "security.ubuntu.com", "archive.ubuntu.com", "azure.archive.ubuntu.com", "motd.ubuntu.com", "*.security.ubuntu.com", "*.archive.ubuntu.com", "entropy.ubuntu.com", "api.snapcraft.io", "changelogs.ubuntu.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "WindowsLanguagePack",
        "priority": 30200,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "WindowsLanguagePack",  "targetFqdns": [ "software-static.download.prss.microsoft.com" ] }
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

az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/firewallPolicies/${TEMP_FWP_NAME}/ruleCollectionGroups/DefaultNetworkRuleCollectionGroup?api-version=2023-05-01" --body @- <<EOF
{
  "properties": {
    "priority": 2000,
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
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "KMS",
        "priority": 30000,
        "rules": [
          { "ruleType": "NetworkRule", "ipProtocols": [ "TCP" ], "sourceAddresses": [ "*" ], "name": "KMS", "destinationPorts": [ 1688 ],  "destinationAddresses": [ "20.118.99.224", "40.83.235.53", "23.102.135.246" ] }
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
