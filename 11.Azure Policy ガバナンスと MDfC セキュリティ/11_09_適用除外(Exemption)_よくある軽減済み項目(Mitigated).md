# 適用除外 (Exemption) : よくある軽減済み項目 (Mitigated)

論理的に問題が起こりえない状況と考えられるために、**当該ルールを適用する必要がない**、と判断される項目について、適用除外処理 (Exemption - Mitigated) を行います。

- フローログ保存用のストレージに対するネットワークセキュリティ確保
- ADE 用の KeyVault に対するネットワークセキュリティ確保
- エージェントの自動プロビジョニング機能の利用の適用
- Application Gateway v2 に割り当てられている Subnet への NSG 適用ルールの除外
- WAF の背後にある Web サーバへのアクセスでは HTTPS 通信を除外
- Private Endpoint サブネットへの UDR 適用の除外
- Application Gateway v2 にシステム的に割り当てられた NSG の診断ログ出力設定を除外

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Azure Security Benchmark'].id" -o tsv)
 
# ■ フローログ保存用のストレージに対するネットワークセキュリティ確保の適用除外 (Mitigated)
# 内部利用のため、プライベートエンドポイントの作成は不要
# Storage accounts should use private link
# /providers/Microsoft.Authorization/policyDefinitions/6edd7eda-6dd8-40f7-810d-67160c639cd9
# storageAccountShouldUseAPrivateLinkConnectionMonitoringEffect
 
TEMP_EXEMPTION_NAME="Exemption-FlowLogStorage"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "storageAccountShouldUseAPrivateLinkConnectionMonitoringEffect"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "フローログ用のストレージであるためプライベートエンドポイントの作成を除外 (Mitigated)",
    "description": "フローログ用のストレージであるためプライベートエンドポイントの作成は不要"
  }
}
EOF
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
  TEMP_RG_NAME="rg-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_FLOWLOG_STORAGE_NAME="stvdcfl${TEMP_LOCATION_PREFIX}${UNIQUE_SUFFIX}"
 
TEMP_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.storage/storageaccounts/${TEMP_FLOWLOG_STORAGE_NAME}"
 
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done
 
# ■ ADE 用の KeyVault に対するネットワークセキュリティ確保の適用除外 (Mitigated)
# 内部利用のため、プライベートエンドポイントの作成は不要
# Azure Key Vaults should use private link
# /providers/Microsoft.Authorization/policyDefinitions/a6abeaec-4d90-4a02-805f-6b26c4d3fbe9
# privateEndpointShouldBeConfiguredForKeyVaultMonitoringEffect

#TEMP_RESOURCE_IDS[1]="/subscriptions/4104fe87-a508-4913-813c-0a23748cd402/resourcegroups/rg-test/providers/microsoft.keyvault/vaults/kv-test-spokea"
#TEMP_RESOURCE_IDS[2]="/subscriptions/903c6183-3adc-4577-9114-b3fef417ff28/resourcegroups/rg-ops-eus/providers/microsoft.keyvault/vaults/kv-ops-ade-20299-eus"

TEMP_EXEMPTION_NAME="Exemption-ADEKeyVault"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "privateEndpointShouldBeConfiguredForKeyVaultMonitoringEffect"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "ADE 用の KeyVault であるため適用を除外 (Mitigated)",
    "description": "ADE 用の KeyVault であるためプライベートエンドポイントの作成は不要"
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

TEMP_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
TEMP_ADE_KV_NAME="kv-hub-ade-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_HUB}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.keyvault/vaults/${TEMP_ADE_KV_NAME}"
j=`expr $j + 1`

done

for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done


# ■ Application Gateway v2 に割り当てられている Subnet には NSG が付与できないため NSG 適用ルールを除外 (Mitigated)
# Subnets should be associated with a Network Security Group
# e71308d3-144b-4262-b144-efdc3cc90517
 
TEMP_EXEMPTION_NAME="Exemption-NSGonAppGatewaySubnet"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "networkSecurityGroupsOnSubnetsMonitoring"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "Application Gateway が利用するサブネットへの NSG 適用の除外 (Mitigated)",
    "description": "Application Gateway が利用するサブネットには NSG が適用できないため"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spokebdmz-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-spokebdmz-${TEMP_LOCATION_PREFIX}"
 
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.network/virtualnetworks/${TEMP_VNET_NAME}/subnets/dmzsubnet"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done
 
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
 
# ■ Private Endpoint サブネットへの UDR 適用除外 (Mitigated)
# （カスタムポリシー用）
 
TEMP_EXEMPTION_NAME="Exemption-UDRonPrivateEndpointSubnet"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "/providers/Microsoft.Management/managementGroups/landingzones/providers/Microsoft.Authorization/policyAssignments/custom-check-lz",
    "policyDefinitionReferenceIds": [
      "custom-policy-check-network-subnet-with-udr"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "Private Endpoint サブネットへの UDR 適用除外 (Mitigated)",
    "description": "Private Endpoint サブネットからの outbound 通信は通常存在しないため。"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-spokeb-${TEMP_LOCATION_PREFIX}"
 
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/${TEMP_RG_NAME}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

# ■ Application Gateway v2 にシステム的に割り当てられた NSG の診断ログ出力設定を除外 (Mitigated)

TEMP_EXEMPTION_NAME="Exemption-DiagnosticsSettingsForNSGonAppGatewaySubnet"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "/providers/Microsoft.Management/managementGroups/landingzones/providers/Microsoft.Authorization/policyAssignments/custom-check-lz",
    "policyDefinitionReferenceIds": [
      "custom-policy-check-monitoring-diagnostic-logs-enabled"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "Application Gateway v2 にシステム的に割り当てられた NSG の診断ログ出力設定を除外 (Mitigated)",
    "description": "AppGateway が利用する NSG はシステム的に管理されているため"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_RG_NAME="rg-spokebdmz-${TEMP_LOCATION_PREFIX}"
TEMP_NSG_NAME="vnet-spokebdmz-${TEMP_LOCATION_PREFIX}-DmzSubnet-nsg-${TEMP_LOCATION_NAME}"

if [ -z $(az network nsg list --resource-group $TEMP_RG_NAME --subscription ${SUBSCRIPTION_ID_SPOKE_B} --query "[? name == \"${TEMP_NSG_NAME}\" ]" -o tsv
) ]; then
  echo "NSG が存在しないため処理を行いません。"
else
  echo "NSG が存在するので処理します。"
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.network/networksecuritygroups/${TEMP_NSG_NAME}"
j=`expr $j + 1`
fi

done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
