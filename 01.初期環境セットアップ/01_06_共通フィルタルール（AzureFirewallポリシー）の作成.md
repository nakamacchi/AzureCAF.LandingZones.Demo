# 共通フィルタルール（Azure Firewall ポリシー）の作成

クラウド環境上では複数の箇所に Azure Firewall を立てることがよくありますが、その際、Windows Update のように、頻繁に利用され、かつ安全性が一定以上確保されているとみなすことができる対象サーバ群については、親ポリシーとしてまとめて管理しておくと便利です。これを定義します。

## (参考) Priority 設計について

以下のように番号を利用しています。

- RuleCollectionGroup
  - 1000 : DefaultApplicationRuleCollectionGroup (親ポリシー側に実装, fwp-vdc-XXX)
    - Application rule collection
      - 20000 : Common Azure Resouces (Source IP = *)
        - 200xx : MDE
          - 20000 : MDE(Streamlined_CoreFunctionality)
          - 20010 : MDE(Streamlined_Updates)
          - 20020 : MDE(Streamlined_CertificateValidationChecks)
          - 20030 : MDE(Streamlined_Others)
          - 20040 : MDE(LinuxOnboarding)
        - 203xx : AMA
        - 204xx : GuestConfiguration
        - 205xx : ApplicationInsights
        - 206xx : GuestAttestation
        - 207xx : Azure Monitor
        - 208xx : Azure Arc
        - 209xx : MDfS (discon)
      - 30000 : Common OS Resources (Source IP = *)
        - 300xx : Windows OS
        - 301xx : Ubuntu OS
  - 2000 : DefaultNetworkRuleCollectionGroup (親ポリシー側に実装, fwp-vdc-XXX)
    - Network rule collection
      - 20000 : Common Azure Resouces (Source IP = *)
        - 200xx : MDE
      - 30000 : Common OS Resources (Source IP = *)
        - 300xx : KMS
        - 301xx : MS_NTP
  - 3000 : AdditionalApplicationRuleCollectionGroup (子ポリシー側に実装, fw-hub/ops-XXX-fwp)
    - Application rule collection
      - 50000 : For LOB Apps (Source IP = Specific Subnet)
        - 500xx : Common 3rd-party Resources (Source IP = *)
          - 50000 : Docker
        - 501xx : Spoke A Apps
        - 502xx : Spoke B Apps
          - 50200 : Spoke_B_AppService(NuGet)
        - 503xx : Spoke C Apps
        - 504xx : Spoke D Apps
        - 505xx : Spoke E Apps
          - 50500 : Spoke_E_AKS
          - 50599 : Spoke_E_Maintanance
        - 506xx : Spoke F Apps
          - 50600 : Spoke_F_ACAEnv
          - 50601 : Spoke_F_MIDAuth
          - 50699 : Spoke_F_Maintanance
      - 60000 : Exception Rules (Source IP = Specific Subnet or IP Address)
        - 600xx : Ops VNET
          - 60000 : Azure Portal
          - 60001 : Azure AI Foundry Portal (AI Studio)
          - 60010 : Internet Access
        - 601xx : Hub VNET
  - 4000 : AdditionalNetworkRuleCollectionGroup (子ポリシー側に実装, fw-hub/ops-XXX-fwp)
    - Network rule collection
      - 50000 : For LOB Apps (Source IP = Specific Subnet)

```bash

# 基盤構築管理者のアカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_plat_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_plat_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
 
# LAW 作成とアクティビティログ出力の設定
az account set -s "${SUBSCRIPTION_ID_MGMT}"

# 親ポリシーの作成

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_FWP_NAME="fwp-vdc-${TEMP_LOCATION_PREFIX}"

TEMP_FWP_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/firewallPolicies/${TEMP_FWP_NAME}"

az rest --method PUT --uri "${TEMP_FWP_ID}?api-version=2021-08-01" --body @- <<EOF
{
  "location": "${TEMP_LOCATION_NAME}",
  "properties": {
    "sku": {
      "tier": "Standard"
    }
  }
}
EOF

while true
do
  HEALTH_STATUS=$(az rest --method GET --uri "${TEMP_FWP_ID}?api-version=2021-08-01" --query properties.provisioningState -o tsv)
  echo "Waiting for provisioning. Current state is $HEALTH_STATUS ..."
  if [ "$HEALTH_STATUS" == "Succeeded" ] || [ "$HEALTH_STATUS" == "Warning" ]; then
    break
  fi
  sleep 10
done

az rest --method PUT --url "${TEMP_FWP_ID}/ruleCollectionGroups/DefaultApplicationRuleCollectionGroup?api-version=2021-08-01" --body @- <<EOF
{
  "location": "${TEMP_LOCATION_NAME}",
  "properties": {
    "priority": 1000,
    "ruleCollections": [
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "MDE(Streamlined_CoreFunctionality)",
        "priority": 20000,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Core Defender for Endpoint services",  "targetFqdns": [ "*.endpoint.security.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Web & network protection",  "targetFqdns": [ "*.smartscreen-prod.microsoft.com", "*.smartscreen.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "SmartScreen",  "targetFqdns": [ "*.smartscreen.microsoft.com", "*.checkappexec.microsoft.com", "*.urs.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Device management through Microsoft Defender for Endpoint security settings management",  "targetFqdns": [ "*.dm.microsoft.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "MDE(Streamlined_Updates)",
        "priority": 20010,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Linux app/platform updates",  "targetFqdns": [ "packages.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Mac app/platform updates",  "targetFqdns": [ "officecdn-microsoft-com.akamaized.net" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Windows/Mac/Linux security intelligence updates, Windows antimalware platform updates, (alternative download location / direct from Defender cloud)",  "targetFqdns": [ "go.microsoft.com", "definitionupdates.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Windows security intelligence and antimalware platform updates, product updates to EDR sensors. (when using Microsoft/Windows update as the source/method)",  "targetFqdns": [ "*.update.microsoft.com", "*.delivery.mp.microsoft.com", "*.windowsupdate.com", "*.download.windowsupdate.com", "*.download.microsoft.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "MDE(Streamlined_CertificateValidationChecks)",
        "priority": 20020,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "Windows operating system certificate validation checks",  "targetFqdns": [ "ctldl.windowsupdate.com", "crl.microsoft.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "MDE(Streamlined_Others)",
        "priority": 20030,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "under www.microsoft.com CRL and downloads",  "targetFqdns": [ "www.microsoft.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "MDE(Linux_Onboarding)",
        "priority": 20040,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "AzureRepository",  "targetFqdns": [ "rhui-1.microsoft.com", "rhui-2.microsoft.com", "rhui-3.microsoft.com", "pakage.microsoft.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "old_MDfS(MDE)",
        "priority": 20900,
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
        "name": "old_MDfS(DefenderAntivirus)",
        "priority": 20901,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "MU / WU",  "targetFqdns": [ "go.microsoft.com", "definitionupdates.microsoft.com", "www.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Symbols",  "targetFqdns": [ "msdl.microsoft.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "MAPS",  "targetFqdns": [ "*.wdcp.microsoft.com", "*.wd.microsoft.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "old_MDfS(Qualys)",
        "priority": 20902,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "QualysCloudScanner",  "targetFqdns": [ "qagpublic.qg3.apps.qualys.com", "qagpublic.qg2.apps.qualys.eu" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "GuestConfigurationAgent",
        "priority": 20400,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "Guest Configuration Agent",  "targetFqdns": [ "agentserviceapi.guestconfiguration.azure.com", "*.guestconfiguration.azure.com", "oaasguestconfigac2s1.blob.core.windows.net", "oaasguestconfigacs1.blob.core.windows.net", "oaasguestconfigaes1.blob.core.windows.net", "oaasguestconfigases1.blob.core.windows.net", "oaasguestconfigbrses1.blob.core.windows.net", "oaasguestconfigbrss1.blob.core.windows.net", "oaasguestconfigccs1.blob.core.windows.net", "oaasguestconfigces1.blob.core.windows.net", "oaasguestconfigcids1.blob.core.windows.net", "oaasguestconfigcuss1.blob.core.windows.net", "oaasguestconfigeaps1.blob.core.windows.net", "oaasguestconfigeas1.blob.core.windows.net", "oaasguestconfigeus2s1.blob.core.windows.net", "oaasguestconfigeuss1.blob.core.windows.net", "oaasguestconfigfcs1.blob.core.windows.net", "oaasguestconfigfss1.blob.core.windows.net", "oaasguestconfiggewcs1.blob.core.windows.net", "oaasguestconfiggns1.blob.core.windows.net", "oaasguestconfiggwcs1.blob.core.windows.net", "oaasguestconfigjiws1.blob.core.windows.net", "oaasguestconfigjpes1.blob.core.windows.net", "oaasguestconfigjpws1.blob.core.windows.net", "oaasguestconfigkcs1.blob.core.windows.net", "oaasguestconfigkss1.blob.core.windows.net", "oaasguestconfigncuss1.blob.core.windows.net", "oaasguestconfignes1.blob.core.windows.net", "oaasguestconfignres1.blob.core.windows.net", "oaasguestconfignrws1.blob.core.windows.net", "oaasguestconfigqacs1.blob.core.windows.net", "oaasguestconfigsans1.blob.core.windows.net", "oaasguestconfigscuss1.blob.core.windows.net", "oaasguestconfigseas1.blob.core.windows.net", "oaasguestconfigsecs1.blob.core.windows.net", "oaasguestconfigsfns1.blob.core.windows.net", "oaasguestconfigsfws1.blob.core.windows.net", "oaasguestconfigsids1.blob.core.windows.net", "oaasguestconfigstzns1.blob.core.windows.net", "oaasguestconfigswcs1.blob.core.windows.net", "oaasguestconfigswns1.blob.core.windows.net", "oaasguestconfigswss1.blob.core.windows.net", "oaasguestconfigswws1.blob.core.windows.net", "oaasguestconfiguaecs1.blob.core.windows.net", "oaasguestconfiguaens1.blob.core.windows.net", "oaasguestconfigukss1.blob.core.windows.net", "oaasguestconfigukws1.blob.core.windows.net", "oaasguestconfigwcuss1.blob.core.windows.net", "oaasguestconfigwes1.blob.core.windows.net", "oaasguestconfigwids1.blob.core.windows.net", "oaasguestconfigwus2s1.blob.core.windows.net", "oaasguestconfigwus3s1.blob.core.windows.net", "oaasguestconfigwuss1.blob.core.windows.net" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "GuestAttestationAgent",
        "priority": 20410,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Guest Attestation Agent",  "targetFqdns": [ "*.attest.azure.net" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "Application Insights",
        "priority": 20500,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Telemetories",  "targetFqdns": [ "dc.applicationinsights.azure.com", "dc.applicationinsights.microsoft.com", "dc.services.visualstudio.com", "*.in.applicationinsights.azure.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Live Metrics",  "targetFqdns": [ "live.applicationinsights.azure.com", "rt.applicationinsights.microsoft.com", "rt.services.visualstudio.com", "*.livediagnostics.monitor.azure.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Profiler",  "targetFqdns": [ "profiler.monitor.azure.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "Azure Monitor",
        "priority": 20700,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Azure Monitor",  "targetFqdns": [ "gcs.prod.monitoring.core.windows.net", "global.prod.microsoftmetrics.com", "shoebox2.prod.microsoftmetrics.com", "shoebox2-red.prod.microsoftmetrics.com", "shoebox2-black.prod.microsoftmetrics.com", "prod3.prod.microsoftmetrics.com", "prod3-black.prod.microsoftmetrics.com", "prod3-red.prod.microsoftmetrics.com", "gcs.prod.warm.ingestion.monitoring.azure.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "Azure Arc",
        "priority": 20800,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Azure Arc-enabled Data Services",  "targetFqdns": [ "*.arcdataservices.com" ] }
        ]
      },      {
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
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "Windows Push Notification",  "targetFqdns": [ "client.wns.windows.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 } ], "sourceAddresses": [ "*" ], "name": "LanguagePackage",  "targetFqdns": [ "software-static.download.prss.microsoft.com" ] }
        ]
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "UbuntuOS",
        "priority": 30100,
        "rules": [
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Https", "port": 443 }, { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "UpdatePackage",  "targetFqdns": [ "security.ubuntu.com", "archive.ubuntu.com", "azure.archive.ubuntu.com", "motd.ubuntu.com", "*.security.ubuntu.com", "*.archive.ubuntu.com", "entropy.ubuntu.com", "api.snapcraft.io", "changelogs.ubuntu.com", "releases.ubuntu.com", "contracts.canonical.com" ] },
          { "ruleType": "ApplicationRule", "protocols": [ { "protocolType": "Http", "port": 80 } ], "sourceAddresses": [ "*" ], "name": "CRL",  "targetFqdns": [ "crl.comodoca.com", "crl.usertrust.com", "ocsp.usertrust.com", "ocsp.comodoca.com", "*.lencr.org" ] }
        ]
      }
    ]
  }
}
EOF

az rest --method PUT --url "${TEMP_FWP_ID}/ruleCollectionGroups/DefaultNetworkRuleCollectionGroup?api-version=2021-08-01" --body @- <<EOF
{
  "location": "${TEMP_LOCATION_NAME}",
  "properties": {
    "priority": 2000,
    "ruleCollections": [
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "MDE",
        "priority": 20000,
        "rules": [
          { "ruleType": "NetworkRule", "ipProtocols": [ "TCP" ], "sourceAddresses": [ "*" ], "name": "MDE", "destinationPorts": [ 80, 443 ],  "destinationAddresses": [ "MicrosoftDefenderForEndpoint", "OneDsCollector" ] }
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
      },
      {
        "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
        "action": { "type": "Allow" },
        "name": "MS_NTP",
        "priority": 30100,
        "rules": [
          { "ruleType": "NetworkRule", "ipProtocols": [ "UDP" ], "sourceAddresses": [ "*" ], "name": "KMS", "destinationPorts": [ 123 ], "destinationAddresses": [ "168.61.215.74" ] }
        ]
      }
    ]
  }
}
EOF

while true
do
  HEALTH_STATUS=$(az rest --method GET --uri "${TEMP_FWP_ID}/ruleCollectionGroups/DefaultApplicationRuleCollectionGroup?api-version=2021-08-01" --query properties.provisioningState -o tsv)
  echo "Waiting for DefaultApplicationRuleCollectionGroup provisioning. Current state is $HEALTH_STATUS ..."
  if [ "$HEALTH_STATUS" == "Succeeded" ] || [ "$HEALTH_STATUS" == "Warning" ]; then
    break
  fi
  sleep 10
done


while true
do
  HEALTH_STATUS=$(az rest --method GET --uri "${TEMP_FWP_ID}/ruleCollectionGroups/DefaultNetworkRuleCollectionGroup?api-version=2021-08-01" --query properties.provisioningState -o tsv)
  echo "Waiting for DefaultNetworkRuleCollectionGroup provisioning. Current state is $HEALTH_STATUS ..."
  if [ "$HEALTH_STATUS" == "Succeeded" ] || [ "$HEALTH_STATUS" == "Warning" ]; then
    break
  fi
  sleep 10
done

# AMA に対する穴あけ
for j in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME2=${LOCATION_NAMES[$j]}
TEMP_LOCATION_PREFIX2=${LOCATION_PREFIXS[$j]}

# LAW, DCE への経路確保
# VDC LAW への FQDN 経路開放（正副両方へ流せるようにしておく）
TEMP_LAW_RES_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX2}/providers/microsoft.operationalinsights/workspaces/law-vdc-${TEMP_LOCATION_PREFIX2}"
TEMP_DCE_RES_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX2}/providers/Microsoft.Insights/dataCollectionEndpoints/dce-vdc-${TEMP_LOCATION_PREFIX2}"
TEMP_LAW_CUSTOMER_ID=$(az monitor log-analytics workspace show --ids ${TEMP_LAW_RES_ID} --query customerId -o tsv)
TEMP_DCE_CONFIG_EP=$(az monitor data-collection endpoint show --ids ${TEMP_DCE_RES_ID} --query configurationAccess.endpoint -o tsv)
TEMP_DCE_CONFIG_FQDN=${TEMP_DCE_CONFIG_EP:8}
TEMP_DCE_INGEST_EP=$(az monitor data-collection endpoint show --ids ${TEMP_DCE_RES_ID} --query logsIngestion.endpoint -o tsv)
TEMP_DCE_INGEST_FQDN=${TEMP_DCE_INGEST_EP:8}

TEMP_COLLECTION_PRIORITY="2030${j}"
 
# 共通ポリシーに対して設定
az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group ${TEMP_RG_NAME} --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
  --name "Azure Monitor Agent" --rule-type ApplicationRule --collection-priority ${TEMP_COLLECTION_PRIORITY} --action Allow \
  --rule-name "${TEMP_LAW_NAME}" \
  --target-fqdns "global.handler.control.monitor.azure.com" "${TEMP_LOCATION_NAME2}.handler.control.monitor.azure.com" "management.azure.com" "${TEMP_LOCATION_NAME2}.monitoring.azure.com" "${TEMP_LAW_CUSTOMER_ID}.ods.opinsights.azure.com" "${TEMP_DCE_CONFIG_FQDN}" "${TEMP_DCE_INGEST_FQDN}" \
  --source-addresses "*" --protocols Http=80 Https=443
done #j

done #i

```
