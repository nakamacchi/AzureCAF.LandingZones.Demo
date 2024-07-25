# Azure Policy (Exemption - Waiver)

Azure Policy の違反項目のうち、**ポリシーの意図が満たされていないが、当該システムではリスクを受容する**ものについて適用を除外します。

- DevCenter 用の KeyVault に対するネットワークファイアウォール機能の有効化（※ 現状未サポートのため）

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Azure Security Benchmark'].id" -o tsv)

# ■ DevCenter 用の KeyVault に対するネットワークファイアウォール機能の有効化（※ 現状未サポートのため） (Waiver)
# Azure Key Vault should have firewall enabled

TEMP_EXEMPTION_NAME="Exemption-DevCenterKeyVaultNetworkFirewall"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "firewallShouldBeEnabledOnKeyVaultMonitoringEffect"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "DevCenter 用の KeyVault に対するファイアウォール有効化が未サポートのため適用を除外 (Waiver)",
    "description": "DevCenter 用の KeyVault に対するファイアウォール有効化が未サポートのため"
  }
}
EOF

TEMP_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourcegroups/rg-devcenter-${LOCATION_PREFIXS[0]}/providers/microsoft.keyvault/vaults/kv-devcenter-${UNIQUE_SUFFIX}-${LOCATION_PREFIXS[0]}"

az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json

```
