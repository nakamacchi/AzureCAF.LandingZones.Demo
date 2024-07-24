# Azure Policy (Exemption - Mitigated)

Azure Policy の違反項目のうち、実質的に違反を起こしていない（ポリシーの意図は満たされている）ものについて、適用を除外します。

- DevCenter 用の KeyVault に対するネットワークセキュリティ確保

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Azure Security Benchmark'].id" -o tsv)

# ■ DevCenter 用の KeyVault に対するネットワークセキュリティ確保の適用除外 (Mitigated)
# 内部利用のため、プライベートエンドポイントの作成は不要
# Azure Key Vaults should use private link
# /providers/Microsoft.Authorization/policyDefinitions/a6abeaec-4d90-4a02-805f-6b26c4d3fbe9
# privateEndpointShouldBeConfiguredForKeyVaultMonitoringEffect

TEMP_EXEMPTION_NAME="Exemption-DevCenterKeyVault"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "privateEndpointShouldBeConfiguredForKeyVaultMonitoringEffect"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "DevCenter 用の KeyVault であるため適用を除外 (Mitigated)",
    "description": "DevCenter 用の KeyVault であるためプライベートエンドポイントの作成は不要"
  }
}
EOF

TEMP_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourcegroups/rg-devcenter-${LOCATION_PREFIXS[0]}/providers/microsoft.keyvault/vaults/kv-devcenter-${UNIQUE_SUFFIX}-${LOCATION_PREFIXS[0]}"

az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json

```
