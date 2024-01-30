# Azure Policy (Exemption - Waiver)

Azure Policy の違反項目のうち、**ポリシーの意図が満たされていないが、当該システムではリスクを受容する**ものについて適用を除外します。

- SQL DB へのアクセスに際しての Azure AD 認証

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Microsoft Cloud Security Benchmark'].id" -o tsv)

# ■ SQL DB へのアクセスに際しての Azure AD 認証の適用除外 (Waiver)
# An Azure Active Directory administrator should be provisioned for SQL servers
# 1f314764-cb73-4fc9-b863-8eca98ac36e9
# aadAuthenticationInSqlServerMonitoring
# Azure SQL Database should have Azure Active Directory Only Authentication enabled
# abda6d70-9778-44e7-84a8-06713e6db027
# sqlServerADOnlyEnabledMonitoring

TEMP_EXEMPTION_NAME="Exemption-AzureADAuthenticationInSQLDatabase"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "aadAuthenticationInSqlServerMonitoring",
      "sqlServerADOnlyEnabledMonitoring"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "SQL DB へのアクセスに関する Azure AD 認証利用の除外 (Waiver)",
    "description": "アプリケーション側での Azure AD 認証の適用が困難なため"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_F}/resourceGroups/rg-spokef-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Sql/servers/sql-spokef-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done

for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

# ■ CSB の適用除外
# Symbolic links が CSB に抵触するため
# Windows machines should meet requirements of the Azure compute security baseline
# /providers/Microsoft.Authorization/policyDefinitions/72650e9f-97bc-4b2a-ab5f-9781a9fcecbc
# windowsGuestConfigBaselinesMonitoring
 
TEMP_EXEMPTION_NAME="Exemption-CSB"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "windowsGuestConfigBaselinesMonitoring"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "VM に対する CSB を免除 (Waiver)",
    "description": "Symbolic links が CSB に抵触するため"
  }
}
EOF
 
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_F}/resourcegroups/rg-spokefmtn-${TEMP_LOCATION_PREFIX}/providers/microsoft.compute/virtualmachines/vm-mnt-${TEMP_LOCATION_PREFIX}"
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
