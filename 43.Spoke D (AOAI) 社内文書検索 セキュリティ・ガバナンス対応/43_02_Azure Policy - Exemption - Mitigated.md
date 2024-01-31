# Azure Policy - Exemption - Mitigated

Azure Policy により Non Compliant となった項目のうち、リスクがない（項目の意図が満たされている）ものについて、Exemption - Mitigated（除外 - 軽減済み）を適用します。

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Microsoft Cloud Security Benchmark'].id" -o tsv)

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

  TEMP_RG_NAME="rg-spokedmtn-${TEMP_LOCATION_PREFIX}"
  TEMP_ADE_KV_NAME="kv-spkdmtn-ade-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"

TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.keyvault/vaults/${TEMP_ADE_KV_NAME}"
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
TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_WEBAPP_NAME="webapp-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
 
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.web/sites/${TEMP_WEBAPP_NAME}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done


# ■ ADE が適用されている VM に対する Host Encryption の適用を除外(Mitigated)

TEMP_EXEMPTION_NAME="Exemption-EncryptionAtHostLevelWithADE"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "virtualMachinesAndVirtualMachineScaleSetsShouldHaveEncryptionAtHostEnabled"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "ADE が適用されている VM に対して EncryptionAtHost の適用を除外 (Mitigated)",
    "description": "ADE が適用されている場合は EncryptionAtHost の適用は不要"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for TEMP_SUBSCRIPTION_ID in ${SUBSCRIPTION_ID_SPOKE_D}; do
az account set -s $TEMP_SUBSCRIPTION_ID
for TEMP_VM_ID in $(az vm list --query [].id -o tsv); do
TEMP_ENC_STATE=$(az vm encryption show --ids $TEMP_VM_ID --query "disks[0].statuses[0].code" -o tsv)
if [[ $TEMP_ENC_STATE == "EncryptionState/encrypted" ]]; then
  echo $TEMP_VM_ID $TEMP_ENC_STATE
  TEMP_RESOURCE_IDS[j]=$TEMP_VM_ID
  j=`expr $j + 1`
fi
done #TEMP_VM_ID
done #TEMP_SUBSCRIPTION_ID

for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
