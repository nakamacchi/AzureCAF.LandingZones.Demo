# 適用除外 (Exemption) : テストでよくある免除項目 (Waiver)

本サンプルのように、デモやテストを目的とする場合、コストやテスト容易性のために、**ルールは充足されていない**が、意識的に無視する項目があります。これらに関して、適用除外処理（Exemption - Waiver）を行います。

なお、特に本ページの適用除外は **実際の本番環境では行うべきものではありません**。デモの都合上、手間がかかりすぎるもの（例：SQL DB の脆弱性対策）や、環境削除作業が面倒になるもの（例：VM Backup の適用）、コストの関係で割愛するもの（例：DDoS Protection Standard や WAF SKU の適用）などを適用除外としています。

- DDoS Protection Standard の適用
- Application Gateway に対する WAF SKU の適用
- VM Backup の適用
- SQL DB の脆弱性対策
- SQL logical Server の脆弱性対策

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
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/rg-spokebdmz-${TEMP_LOCATION_PREFIX}/providers/microsoft.network/virtualnetworks/vnet-spokebdmz-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-ops-${TEMP_LOCATION_PREFIX}/providers/microsoft.network/virtualnetworks/vnet-ops-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourcegroups/rg-hub-${TEMP_LOCATION_PREFIX}/providers/microsoft.network/virtualnetworks/vnet-hub-${TEMP_LOCATION_PREFIX}"
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
 
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/rg-spokebdmz-${TEMP_LOCATION_PREFIX}/providers/microsoft.network/applicationgateways/waf-spokebdmz-${TEMP_LOCATION_PREFIX}"
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
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-ops-${TEMP_LOCATION_PREFIX}/providers/microsoft.compute/virtualmachines/vm-ops-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourcegroups/rg-hub-${TEMP_LOCATION_PREFIX}/providers/microsoft.compute/virtualmachines/vm-usr-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done
 
# ■ サンプルアプリのため、SQL DB 関連のセキュリティ対策を免除
# SQL databases should have vulnerability findings resolved
# /providers/Microsoft.Authorization/policyDefinitions/feedbf84-6b99-488c-acc2-71c829aa5ffc
# sqlDbVulnerabilityAssesmentMonitoring

TEMP_EXEMPTION_NAME="Test-Exemption-SQLDBVulnerabilityAssessment"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "sqlDbVulnerabilityAssesmentMonitoring"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "テスト目的での免除 - SQL DB の構成脆弱性排除の適用免除",
    "description": "テスト目的での免除 - SQL DB の構成脆弱性排除"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/rg-spokeb-${TEMP_LOCATION_PREFIX}/providers/microsoft.sql/servers/sql-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
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
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/rg-spokeb-${TEMP_LOCATION_PREFIX}/providers/microsoft.sql/servers/sql-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
