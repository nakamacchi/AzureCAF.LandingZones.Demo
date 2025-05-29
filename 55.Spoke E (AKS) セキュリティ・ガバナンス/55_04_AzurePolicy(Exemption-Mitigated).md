# Azure Policy (Exemption - Mitigated)

Azure Policy の違反項目のうち、実質的に違反を起こしていない（ポリシーの意図は満たされている）ものについて、適用を除外します。

- SQL databases should have vulnerability findings resolved (Policy ID feedbf84-6b99-488c-acc2-71c829aa5ffc, MDfC ID 82e20e14-edc5-4373-bfc4-f13121257c37)
  - SQL DB に対する構成脆弱性を確認すると以下 3 つの項目が報告されますが、いずれも問題はありません。
    - VA2130 Track all users with access to the database : MID 2 つにアクセス権限を与えていますが、いずれも正しい付与であり、問題はありません。
    - VA2109 Minimal set of principals should be members of fixed low impact database roles : 前述の MID に対して pubs DB の db_datareader/writer ロールを割り当てていますが、いずれも適切なアクセス権限付与であり、問題はありません。

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Microsoft Cloud Security Benchmark'].id" -o tsv)

# ■ 報告されている SQL DB の構成脆弱性は問題がないため、Mitigated 扱いにする
# SQL databases should have vulnerability findings resolved
# /providers/Microsoft.Authorization/policyDefinitions/feedbf84-6b99-488c-acc2-71c829aa5ffc
# sqlDbVulnerabilityAssesmentMonitoring

TEMP_EXEMPTION_NAME="Exemption-SQLDBVulnerabilityAssessment"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "sqlDbVulnerabilityAssesmentMonitoring"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "SQL DB の構成脆弱性排除の適用免除",
    "description": "報告されている脆弱性に問題がない"
  }
}
EOF

TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourcegroups/rg-spokee-${TEMP_LOCATION_PREFIX}/providers/microsoft.sql/servers/sql-spokee-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done

for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```

### (参考) 以前あったが今は不要になったもの

以下は 2024/9/7 に deprecated 扱いになり、不要になりました。

- Vulnerabilities in container security configurations should be remediated (e8cbc669-f12d-49eb-93e7-9273119e9933) (containerBenchmarkMonitoring)
- Vulnerabilities in security configuration on your virtual machine scale sets should be remediated (3c735d8a-a4ba-4a3a-b7cf-db7754cf57f4) (vmssOsVulnerabilitiesMonitoring)
  - これらのポリシーは AKS の VMSS に対しても評価されますが、MDE から情報が通知されないため、Azure Policy としては Non-Compliant 扱いになってしまいます。
  - これらの項目は AKS 基盤側で制御されているため、Mitigated 扱いにします。

```bash

# AKS ノードプールに対する以下のセキュリティ評価を無効化
# Vulnerabilities in container security configurations should be remediated (e8cbc669-f12d-49eb-93e7-9273119e9933) (containerBenchmarkMonitoring)
# Vulnerabilities in security configuration on your virtual machine scale sets should be remediated (3c735d8a-a4ba-4a3a-b7cf-db7754cf57f4) (vmssOsVulnerabilitiesMonitoring)

TEMP_EXEMPTION_NAME="Exemption-AKSvmss"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "containerBenchmarkMonitoring",
      "vmssOsVulnerabilitiesMonitoring"
    ],
    "exemptionCategory": "Mitigated",
    "displayName": "AKS VMSS に対する脆弱性評価の適用免除",
    "description": "MDE からデータが報告されないため"
  }
}
EOF

TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
TEMP_RG_NAME="rg-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_AKS_CLUSTER_NAME="aks-spokee-${TEMP_LOCATION_PREFIX}"
TEMP_NODE_RG_NAME=$(az aks show --name ${TEMP_AKS_CLUSTER_NAME} --resource-group ${TEMP_RG_NAME} --subscription ${SUBSCRIPTION_ID_SPOKE_E} --query nodeResourceGroup -o tsv)

TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourcegroups/${TEMP_NODE_RG_NAME}"
j=`expr $j + 1`
done #i

for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
