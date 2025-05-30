# Azure Policy (Exemption - Waiver)

Azure Policy の違反項目のうち、**ポリシーの意図が満たされていないが、当該システムではリスクを受容する**ものについて適用を除外します。

55_02_k8s 用 Azure Policy の調整にて解説した通り、以下 4 つのポリシーの適用を除外します。

- ASP.NET Core Blazor アプリのための除外

| MDfC Recommendation | MDfC ID | Azure Policy | Policy ID |
| --- | --- | --- | --- |
| Running containers as root user should be avoided | 9b795646-9130-41a4-90b7-df9eae2437c8 | Kubernetes cluster pods and containers should only run with approved user and group IDs | f06ddb64-5fa3-4b77-b166-acb36f7f6042 |
| Immutable (read-only) root filesystem should be enforced for containers | 27d6f0e9-b4d5-468b-ae7e-03d5473fd864 | Kubernetes cluster containers should run with a read only root file system | df49d893-a74c-421d-bc95-c663042e5b80 |

- k8s dashboard のための除外

| MDfC Recommendation | MDfC ID | Azure Policy | Policy ID |
| --- | --- | --- | --- |
| Container CPU and memory limits should be enforced | 405c9ae6-49f9-46c4-8873-a86690f27818 | Kubernetes cluster containers CPU and memory resource limits should not exceed the specified limits | e345eecc-fa47-480f-9e88-67dcc122b164 |
| Kubernetes clusters should disable automounting API credentials| | 32060ac3-f17f-4848-db8e-e7cf2c9a53eb | Kubernetes clusters should disable automounting API credentials | 423dd1ba-798e-40e4-9c4d-b6902674b423 |

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ■ 以下は全体に共通
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
TEMP_ASSIGNMENT_ID=$(az policy assignment list --scope $TEMP_MG_TRG_ID --query "[? displayName == 'Microsoft Cloud Security Benchmark'].id" -o tsv)

# ASP.NET Core Blazor アプリのための除外
# MustRunAsNonRoot f06ddb64-5fa3-4b77-b166-acb36f7f6042
# ReadOnlyRootFileSystemInKubernetesCluster df49d893-a74c-421d-bc95-c663042e5b80

TEMP_EXEMPTION_NAME="Exemption-SecureContainerConfigurationForASP.NET"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "MustRunAsNonRoot",
      "ReadOnlyRootFileSystemInKubernetesCluster"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "ASP.NET Blazor アプリのコンテナセキュリティ構成に関する除外 (Waiver)",
    "description": "セキュリティ構成が困難なため"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourcegroups/rg-spokee-${TEMP_LOCATION_PREFIX}/providers/microsoft.containerservice/managedclusters/aks-spokee-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

# k8s Dashboard のための除外
# memoryAndCPULimitsInKubernetesCluster e345eecc-fa47-480f-9e88-67dcc122b164
# KubernetesClustersShouldDisableAutomountingAPICredentialsMonitoringEffect 423dd1ba-798e-40e4-9c4d-b6902674b423

TEMP_EXEMPTION_NAME="Exemption-SecureContainerConfigurationForK8sDashboard"
cat > temp.json << EOF
{
  "properties": {
    "policyAssignmentId": "${TEMP_ASSIGNMENT_ID}",
    "policyDefinitionReferenceIds": [
      "memoryAndCPULimitsInKubernetesCluster",
      "KubernetesClustersShouldDisableAutomountingAPICredentialsMonitoringEffect"
    ],
    "exemptionCategory": "Waiver",
    "displayName": "k8s Dashboard のコンテナセキュリティ構成に関する除外 (Waiver)",
    "description": "セキュリティ構成が困難なため"
  }
}
EOF
 
TEMP_RESOURCE_IDS=()
j=0
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
TEMP_RESOURCE_IDS[j]="/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}/resourcegroups/rg-spokee-${TEMP_LOCATION_PREFIX}/providers/microsoft.containerservice/managedclusters/aks-spokee-${TEMP_LOCATION_PREFIX}"
j=`expr $j + 1`
done
 
for TEMP_RESOURCE_ID in ${TEMP_RESOURCE_IDS[@]}; do
az rest --method PUT --uri "${TEMP_RESOURCE_ID}/providers/Microsoft.Authorization/policyExemptions/${TEMP_EXEMPTION_NAME}?api-version=2022-07-01-preview" --body @temp.json
done

```
