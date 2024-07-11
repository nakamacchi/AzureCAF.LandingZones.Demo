# 適用除外 (Exemption) : よくある免除項目 (Waiver)

**ルールは充足されていない**が、それによって生じるリスクを許容する、という場合には Waiver による適用免除を行います。

ここでは以下の項目について、Waiver として適用を除外します。

- App Service のクライアント証明書を推奨するポリシー
  - 今回のサンプルでは認証を入れるとテストがしにくいため。
- SQL DB へのアクセスに際しての Azure AD 認証
  - アプリケーション側での Azure AD 認証利用が困難なため。
- CSB の適用除外
  - IIS, SQL などのミドルウェアをセットアップすると CSB 標準からは外れる部分が出るため。CSB の中の一部の項目のみ適用除外とするのが正しいですが、細かい設定ができないため、ここでは CSB をまるごと適用除外としています。
- Cloud Shell で利用するストレージに対するセキュリティ強化ポリシー
  - 非常に便利な機能ですが、本番環境では Cloud Shell は利用しない方がよいでしょう。

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Microsoft Cloud Security Benchmark'].id" -o tsv)

# ■ App Service のクライアント証明書を推奨するポリシーを除外 (Waiver)
#[Deprecated]: App Service apps should have 'Client Certificates (Incoming client certificates)' enabled (5bb220d9-2698-4ee4-8404-b9c30c9df609)
#ensureWEBAppHasClientCertificatesIncomingClientCertificatesSetToOnMonitoringEffect
#App Service apps should have Client Certificates (Incoming client certificates) enabled (19dd1db6-f442-49cf-a838-b0786b4401ef)
#ensureWebAppHasIncomingClientCertificatesSetToOnMonitoringEffect
#TEMP_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/rg-spokeb-eus/providers/microsoft.web/sites/webapp-spokeb-eus"
 
TEMP_EXEMPTION_NAME="Exemption-AppServiceClientCertificates"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "ensureWebAppHasIncomingClientCertificatesSetToOnMonitoringEffect"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "AppServiceのクライアント証明書認証を除外 (Waiver)",
    "description": "今回のサンプルでは認証を入れるとテストがしにくいため"
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
 
TEMP_EXEMPTION_NAME="Exemption-CSB"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "windowsGuestConfigBaselinesMonitoring"
    ],
    "exemptionCategory": "Waiver",
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


# ■ Cloud Shell ストレージアカウントへのセキュリティ強化ルールの適用除外 (Waiver)
 
TEMP_EXEMPTION_NAME="Exemption-StorageAccountForCloudShell"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "storageAccountsShouldRestrictNetworkAccessUsingVirtualNetworkRulesMonitoringEffect",
      "storageAccountShouldUseAPrivateLinkConnectionMonitoringEffect",
      "storageAccountsShouldPreventSharedKeyAccess"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "Cloud Shell ストレージアカウントへのセキュリティ強化ルールの適用除外 (Waiver)",
    "description": "PoC 環境のため"
  }
}
EOF

for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_IDS; do
az account set -s $TEMP_SUBSCRIPTION_ID
for TEMP_RG_NAME in $(az group list --query "[?starts_with(name,'cloud-shell-storage-')].name" -o tsv); do
for TEMP_RESOURCE_ID in $(az storage account list --resource-group $TEMP_RG_NAME --query "[].id" -o tsv); do
echo $TEMP_RESOURCE_ID
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json

done #TEMP_STORAGE_ID
done #TEMP_RG_NAME
done #TEMP_SUBSCRIPTION_ID

```
