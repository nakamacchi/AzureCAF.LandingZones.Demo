# Azure Policy (Exemption - Waiver - Test/PoC)

Azure Policy の違反項目のうち、**ポリシーの意図が満たされていないが、テスト・検証作業であるためにコストが見合わない、作業の手間が増えすぎるなどの理由で、リスクを受容する**ものについて適用を除外します。

- DDoS Protection 適用
- WAF 適用
- VM Backup 適用

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Microsoft Cloud Security Benchmark'].id" -o tsv)

# ■ DDoS Protection 適用の免除（コストのため）

TEMP_EXEMPTION_NAME="Test-Exemption-DDoSProtection"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "vnetEnableDDoSProtectionMonitoring"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "テスト目的での免除 - DDoS Protection の適用免除",
    "description": "テスト目的での免除 - DDoS Protection の適用免除"
  }
}
EOF

TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_F}/resourcegroups/rg-spokefdmz-${TEMP_LOCATION_PREFIX}/providers/microsoft.network/virtualnetworks/vnet-spokefdmz-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_F}/resourcegroups/rg-spokef-${TEMP_LOCATION_PREFIX}/providers/microsoft.network/virtualnetworks/vnet-spokef-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done

for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

# ■ WAF 適用の免除
# Web Application Firewall (WAF) should be enabled for Application Gateway (564feb30-bf6a-4854-b4bb-0d2d2d1e6c66)

TEMP_EXEMPTION_NAME="Test-Exemption-WAFSKU"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "webApplicationFirewallShouldBeEnabledForApplicationGatewayMonitoringEffect"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "テスト目的での免除 - WAF SKU の適用免除",
    "description": "テスト目的での免除 - WAF SKU の適用免除"
  }
}
EOF

TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_F}/resourcegroups/rg-spokefdmz-${TEMP_LOCATION_PREFIX}/providers/microsoft.network/applicationgateways/waf-spokefdmz-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done

for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

# ■ VM Backup の適用免除
# Azure Backup should be enabled for Virtual Machines
# /providers/Microsoft.Authorization/policyDefinitions/013e242c-8828-4970-87b3-ab247555486d
# azureBackupShouldBeEnabledForVirtualMachinesMonitoringEffect

TEMP_EXEMPTION_NAME="Test-Exemption-VMBackup"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "azureBackupShouldBeEnabledForVirtualMachinesMonitoringEffect"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "テスト目的での免除 - VM Backup の適用免除",
    "description": "テスト目的での免除 - VM Backup の適用免除"
  }
}
EOF

TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_F}/resourcegroups/rg-spokefmtn-${TEMP_LOCATION_PREFIX}/providers/microsoft.compute/virtualmachines/vm-mtn-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done

for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

# ■ サンプルアプリのため、SQL Server 関連のセキュリティ対策を免除
# Vulnerability assessment should be enabled on your SQL servers
# /providers/Microsoft.Authorization/policyDefinitions/ef2a8f2a-b3d9-49cd-a8a8-9a3aaaf647d9
# vulnerabilityAssessmentOnServerMonitoring

TEMP_EXEMPTION_NAME="Test-Exemption-SQLServerVulnerabilityAssessment"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "vulnerabilityAssessmentOnServerMonitoring"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "テスト目的での免除 - SQL Server の構成脆弱性排除の適用免除",
    "description": "テスト目的での免除 - SQL Server の構成脆弱性排除"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_F}/resourcegroups/rg-spokef-${TEMP_LOCATION_PREFIX}/providers/microsoft.sql/servers/sql-spokef-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
