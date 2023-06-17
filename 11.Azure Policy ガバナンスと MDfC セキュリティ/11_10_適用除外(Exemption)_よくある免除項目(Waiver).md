# 適用除外 (Exemption) : よくある免除項目 (Waiver)

**ルールは充足されていない**が、それによって生じるリスクを許容する、という場合には Waiver による適用免除を行います。

ここでは以下の項目について、Waiver として適用を除外します。

- App Service のクライアント証明書を推奨するポリシー
  - 認証機能がそもそも不要なアプリであるため。
- SQL DB へのアクセスに際しての Azure AD 認証
  - アプリケーション側での Azure AD 認証利用が困難なため。
- CSB の適用除外
  - IIS, SQL などのミドルウェアをセットアップすると CSB 標準からは外れる部分が出るため。CSB の中の一部の項目のみ適用除外とするのが正しいですが、細かい設定ができないため、ここでは CSB をまるごと適用除外としています。

```bash

# ■ App Service のクライアント証明書を推奨するポリシーを除外 (Waiver)
# App Service apps should have 'Client Certificates (Incoming client certificates)' enabled
#TEMP_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/rg-spokeb-eus/providers/microsoft.web/sites/webapp-spokeb-eus"
 
TEMP_EXEMPTION_NAME="Exemption-AppServiceClientCertificates"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "ensureWEBAppHasClientCertificatesIncomingClientCertificatesSetToOnMonitoringEffect"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "AppServiceのクライアント証明書認証を除外 (Waiver)",
    "description": "認証機能がそもそも不要なアプリであるため"
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
    "exemptionCategory": "Mitigated",
    "displayName": "SQL DB へのアクセスに関する Azure AD 認証利用の除外 (Waiver)",
    "description": "アプリケーション側での Azure AD 認証の適用が困難なため"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_SQL_SERVER_NAME="sql-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_SQL_SERVER_NAME="sql-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
 
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Sql/servers/${TEMP_SQL_SERVER_NAME}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

# ■ CSB の適用除外
# ミドルウェア（IIS, SQL Server など）による設定変更が CSB ルールに抵触するため除外
# Windows machines should meet requirements of the Azure compute security baseline
# /providers/Microsoft.Authorization/policyDefinitions/72650e9f-97bc-4b2a-ab5f-9781a9fcecbc
# windowsGuestConfigBaselinesMonitoring
 
TEMP_EXEMPTION_NAME="Exemption-FlowLogStorage"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "windowsGuestConfigBaselinesMonitoring"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "ミドルウェア (IIS, SQL) が CSB に抵触するため適用を免除 (Waiver)",
    "description": "いったん CSB によるハードニングを行った上でミドルウェアをインストールしているため、他の項目については充足されている"
  }
}
EOF
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
TEMP_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_A}/resourcegroups/rg-spokea-${TEMP_LOCATION_PREFIX}/providers/microsoft.compute/virtualmachines/vm-db-${TEMP_LOCATION_PREFIX}"
 
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
 
TEMP_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_A}/resourcegroups/rg-spokea-${TEMP_LOCATION_PREFIX}/providers/microsoft.compute/virtualmachines/vm-web-${TEMP_LOCATION_PREFIX}"
 
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
 
done

```
