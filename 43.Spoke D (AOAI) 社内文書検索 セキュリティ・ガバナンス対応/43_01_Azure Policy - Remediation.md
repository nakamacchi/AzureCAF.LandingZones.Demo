# Azure Policy - Remediation

Azure Policy により Non Compliant となった項目のうち、是正が可能な項目を是正します。

- Cosmos DB のローカル認証を無効化
- AOAI のローカル認証を無効化
- Form Recognizer のローカル認証を無効化
  - disableLocalAuth を false に変更します。[詳細](https://docs.microsoft.com/azure/cosmos-db/how-to-setup-rbac#disable-local-auth)
- Web Apps の HTTPS を強制
- CSB 違反
  - vm-mtn-xxx には CSB を適用しましたが、その後にソフトウェアなどをインストールすると、CSB 準拠状態が崩れることがあります。
  - WSL2 のインストールにより、下記の CSB 項目が non-compliant になりますので、改めて是正して compliant 状態に戻します。
    - Windows Server must be configured to prevent Internet Control Message Protocol (ICMP) redirects from overriding Open Shortest Path First (OSPF)-generated routes.
      - SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\EnableICMPRedirect を 0 にすることで解決できる
      - PowerShell の場合は Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "EnableICMPRedirect" -Value 0

```bash

# 業務システム D チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spoked_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
 
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_COSMOSDB_NAME="cos-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_COSMOSDB_SQLDB_NAME="ChatHistory"
TEMP_COSMOSDB_CONTAINER_NAME="Prompts"

# ローカルキー認証を無効化
TEMP_COSMOSDB_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/rg-spoked-${TEMP_LOCATION_PREFIX}/providers/Microsoft.DocumentDB/databaseAccounts/${TEMP_COSMOSDB_NAME}"
az rest --method patch --uri "${TEMP_COSMOSDB_ID}?api-version=2023-04-15" --body @- <<EOF
{
    "properties": {
        "disableLocalAuth": true
    }
}
EOF

TEMP_AOAI_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/rg-spoked-${TEMP_LOCATION_PREFIX}/providers/Microsoft.CognitiveServices/accounts/aoai-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
az rest --method patch --uri "${TEMP_AOAI_ID}?api-version=2023-05-01" --body @- <<EOF
{
    "properties": {
        "disableLocalAuth": true
    }
}
EOF

TEMP_FORMRECOGNIZER_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/rg-spoked-${TEMP_LOCATION_PREFIX}/providers/Microsoft.CognitiveServices/accounts/fmr-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
az rest --method patch --uri "${TEMP_FORMRECOGNIZER_ID}?api-version=2023-05-01" --body @- <<EOF
{
    "properties": {
        "disableLocalAuth": true
    }
}
EOF

done # TEMP_LOCATION

# Web Apps の HTTPS を強制
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_WEBAPP_NAME="webapp-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
az webapp update --resource-group ${TEMP_RG_NAME} --name ${TEMP_WEBAPP_NAME} --set httpsOnly=true --subscription ${SUBSCRIPTION_ID_SPOKE_D}
done # TEMP_LOCATION

# CSB の再是正
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_VM_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/rg-spokedmtn-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Compute/virtualMachines/vm-mtn-${TEMP_LOCATION_PREFIX}"
az rest --method post --url "https://management.azure.com${TEMP_VM_ID}/runCommand?api-version=2023-03-01" --body "{\"commandId\":\"RunPowerShellScript\",\"script\":[\"Set-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Services\\Tcpip\\Parameters' -Name 'EnableICMPRedirect' -Value 0\"]}"
done # TEMP_LOCATION

```
