# Azure Policy (Exemption - Mitigated)

Azure Policy の違反項目のうち、実質的に違反を起こしていない（ポリシーの意図は満たされている）ものについて、適用を除外します。

- ADE 用の KeyVault に対するネットワークセキュリティ確保

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi

# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Azure Security Benchmark'].id" -o tsv)

# ■ ADE 用の KeyVault に対するネットワークセキュリティ確保の適用除外 (Mitigated)
# 内部利用のため、プライベートエンドポイントの作成は不要
# Azure Key Vaults should use private link
# /providers/Microsoft.Authorization/policyDefinitions/a6abeaec-4d90-4a02-805f-6b26c4d3fbe9
# privateEndpointShouldBeConfiguredForKeyVaultMonitoringEffect

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
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_F}/resourcegroups/rg-spokefmtn-${TEMP_LOCATION_PREFIX}/providers/microsoft.keyvault/vaults/kv-spkfmtn-ade-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done

for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
