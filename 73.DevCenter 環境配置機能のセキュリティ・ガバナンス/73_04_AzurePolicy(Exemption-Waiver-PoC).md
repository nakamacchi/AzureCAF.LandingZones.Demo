# Azure Policy (Exemption - Waiver - Test/PoC)

Azure Policy の違反項目のうち、**ポリシーの意図が満たされていないが、テスト・検証作業であるためにコストが見合わない、作業の手間が増えすぎるなどの理由で、リスクを受容する**ものについて適用を除外します。

- DevCenter 用の KeyVault に対する削除保護

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Azure Security Benchmark'].id" -o tsv)

# ■ DevCenter 用の KeyVault に対する削除保護の適用免除
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
    "displayName": "テスト目的での免除 - DevCenter 用の KeyVault に対する削除保護の適用免除",
    "description": "テスト目的での免除 - DevCenter 用の KeyVault に対する削除保護の適用免除"
  }
}
EOF

TEMP_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourcegroups/rg-devcenter-${LOCATION_PREFIXS[0]}/providers/microsoft.keyvault/vaults/kv-devcenter-${UNIQUE_SUFFIX}-${LOCATION_PREFIXS[0]}"
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json

```
