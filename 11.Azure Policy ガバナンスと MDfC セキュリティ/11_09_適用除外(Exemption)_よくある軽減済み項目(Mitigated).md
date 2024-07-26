# 適用除外 (Exemption) : よくある軽減済み項目 (Mitigated)

論理的に問題が起こりえない状況と考えられるために、**当該ルールを適用する必要がない**、と判断される項目について、適用除外処理 (Exemption - Mitigated) を行います。

- WAF の背後にある Web サーバへのアクセスでは HTTPS 通信を除外

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
 
# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Microsoft Cloud Security Benchmark'].id" -o tsv)

# ■ WAF の背後にある Web サーバへのアクセスでは HTTPS 通信を除外 (Mitigated)
# App Service apps should only be accessible over HTTPS
# a4af4a39-4135-47fb-b175-47fbdf85311d
# webAppEnforceHttpsMonitoring

TEMP_EXEMPTION_NAME="Exemption-webAppEnforceHttpsMonitoring"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "webAppEnforceHttpsMonitoring"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "WAF の背後にある Web サーバであるため、HTTPS 適用を除外 (Mitigated)",
    "description": "WAF の背後にある Web サーバであり、内部通信については暗号化不要"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_WEBAPP_NAME="webapp-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
 
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.web/sites/${TEMP_WEBAPP_NAME}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
