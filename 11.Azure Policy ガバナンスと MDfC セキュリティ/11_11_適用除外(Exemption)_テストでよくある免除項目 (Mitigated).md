# 適用除外 (Exemption) : テストでよくある免除項目 (Waiver)

本サンプルのように、デモやテストを目的とする場合、コストやテスト容易性のために、**ルールは充足されていない**が、意識的に無視する項目があります。これらに関して、適用除外処理（Exemption - Waiver）を行います。

なお、特に本ページの適用除外は **実際の本番環境では行うべきものではありません**。デモの都合上、手間がかかりすぎるもの（例：MFA 適用や SQL DB の脆弱性対策）や、環境削除作業が面倒になるもの（例：ADE KeyVault の削除保護や VM Backup の適用）、コストの関係で割愛するもの（例：DDoS Protection Standard の適用）などを適用除外としています。

- 作業アカウントに対する MFA の適用
- DDoS Protection Standard の適用
- ADE 用の KeyVault に対する削除保護の適用
- Application Gateway に対する WAF SKU の適用
- VM Backup の適用
- SQL DB の脆弱性対策
- SQL logical Server の脆弱性対策
- IaaS SQL VM 関連の脆弱性対策

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Azure Security Benchmark'].id" -o tsv)
 
# ■ MFA 適用を免除（※ テスト目的の場合のみ）
# Accounts with write permissions on Azure resources should be MFA enabled
# /providers/Microsoft.Authorization/policyDefinitions/931e118d-50a1-4457-a5e4-78550e086c52
# identityEnableMFAForWritePermissionsMonitoringEffect
# Accounts with owner permissions on Azure resources should be MFA enabled
# /providers/Microsoft.Authorization/policyDefinitions/e3e008c3-56b9-4133-8fd7-d3347377402a
# identityEnableMFAForOwnerPermissionsMonitoringNew
# MFA should be enabled on accounts with owner permissions on your subscription
# /providers/Microsoft.Authorization/policyDefinitions/aa633080-8b72-40c4-a2d7-d00c03e80bed
# identityEnableMFAForOwnerPermissionsMonitoring
# MFA should be enabled for accounts with write permissions on your subscription
# /providers/Microsoft.Authorization/policyDefinitions/9297c21d-2ed6-4474-b48f-163f75654ce3
# identityEnableMFAForWritePermissionsMonitoring
# Accounts with read permissions on Azure resources should be MFA enabled
# /providers/Microsoft.Authorization/policyDefinitions/81b3ccb4-e6e8-4e4a-8d05-5df25cd29fd4
# identityEnableMFAForReadPermissionsMonitoringNew
# MFA should be enabled on accounts with read permissions on your subscription
# /providers/Microsoft.Authorization/policyDefinitions/e3576e28-8b17-4677-84c3-db2990658d64
# identityEnableMFAForReadPermissionsMonitoring
# 以下 3 つのポリシーは 2023/05/04 の変更で除去されているので除外
#      "identityEnableMFAForOwnerPermissionsMonitoring",
#      "identityEnableMFAForWritePermissionsMonitoring",
#      "identityEnableMFAForReadPermissionsMonitoring",
 
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "identityEnableMFAForOwnerPermissionsMonitoringNew",   
      "identityEnableMFAForWritePermissionsMonitoringNew", 
      "identityEnableMFAForReadPermissionsMonitoringNew"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "テスト目的での免除 - MFA を免除",
    "description": "テスト目的での免除 - MFA を免除"
  }
}
EOF
 
az rest --method PUT --uri "${TEMP_MG_TRG_ID}/providers/Microsoft.Authorization/policyExemptions/Test-MFA-Exemption?api-version=2022-07-01-preview" --body @temp.json
 
# ■ ADE 用の KeyVault に対する削除保護の適用免除
# Key vaults should have deletion protection enabled
# /providers/Microsoft.Authorization/policyDefinitions/0b60c0b2-2dc2-4e1c-b5c9-abbed971de53
# keyVaultsShouldHavePurgeProtectionEnabledMonitoringEffect
 
TEMP_EXEMPTION_NAME="Test-Exemption-KeyVaultPurgeProtection"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "keyVaultsShouldHavePurgeProtectionEnabledMonitoringEffect"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "テスト目的での免除 - ADE 用の KeyVault に対する削除保護の適用免除",
    "description": "テスト目的での免除 - ADE 用の KeyVault に対する削除保護の適用免除"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
  TEMP_RG_NAME="rg-spokea-${TEMP_LOCATION_PREFIX}"
  TEMP_ADE_KV_NAME="kv-spokea-ade-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_A}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.keyvault/vaults/${TEMP_ADE_KV_NAME}"
j=`expr $j + 1`
 
  TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
TEMP_ADE_KV_NAME="kv-ops-ade-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.keyvault/vaults/${TEMP_ADE_KV_NAME}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done
 
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
 
 
# ■ サンプルアプリのため、IaaS SQL VM 関連のセキュリティ対策を免除
# ★ SQL VM がインストールされていない PC に対しても non-compliant が出てしまうので何かしら不具合がある可能性がある
 
# SQL servers on machines should have vulnerability findings resolved
# /providers/Microsoft.Authorization/policyDefinitions/6ba6d016-e7c3-4842-b8f2-4992ebc0d72d
# serverSqlDbVulnerabilityAssesmentMonitoring
 
TEMP_EXEMPTION_NAME="Test-Exemption-SQLDBVulnerabilityAssessment"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "serverSqlDbVulnerabilityAssesmentMonitoring"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "テスト目的での免除 - IaaS SQL VM の構成脆弱性排除の適用免除",
    "description": "テスト目的での免除 - IaaS SQL VM の構成脆弱性排除"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_A}/resourcegroups/rg-spokea-${TEMP_LOCATION_PREFIX}/providers/microsoft.compute/virtualmachines/vm-db-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_A}/resourcegroups/rg-spokea-${TEMP_LOCATION_PREFIX}/providers/microsoft.compute/virtualmachines/vm-web-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-ops-${TEMP_LOCATION_PREFIX}/providers/microsoft.compute/virtualmachines/vm-ops-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
 
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
