# Azure Policy によるセキュリティ強化

前項では、Azure Policy によるリソースの作成制限を示しましたが、他にも、管理がずさんになりやすいセキュリティについて強制的に有効化するような設定を Azure Policy (Modify/DINE ポリシー) により設定することができます。

## Modify/DINE ポリシーによるセキュリティ強化

[Microsoft Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/policy-reference#microsoft-defender-for-cloud-category) カテゴリーの Azure Policy には、セキュリティ機能を強制的に有効化するようなものが多数含まれています。例えば以下のようなポリシーは、サンドボックス系サブスクリプションを管理する管理グループレベルで有効化してしまうと安心です。

| Policy ID | Policy Name |
| ---- | ---- |
| b99b73e7-074b-4089-9395-b7236f094491 | [Configure Azure Defender for Azure SQL database to be enabled](https://www.azadvertizer.net/azpolicyadvertizer/b99b73e7-074b-4089-9395-b7236f094491.html) |
| b40e7bcd-a1e5-47fe-b9cf-2f534d0bfb7d | [Configure Azure Defender for App Service to be enabled](https://www.azadvertizer.net/azpolicyadvertizer/b40e7bcd-a1e5-47fe-b9cf-2f534d0bfb7d.html) |
| 1f725891-01c0-420a-9059-4fa46cb770b7 | [Configure Azure Defender for Key Vaults to be enabled](https://www.azadvertizer.net/azpolicyadvertizer/1f725891-01c0-420a-9059-4fa46cb770b7.html) |
| b7021b2b-08fd-4dc0-9de7-3c6ece09faf9 | [Configure Azure Defender for Resource Manager to be enabled](https://www.azadvertizer.net/azpolicyadvertizer/b7021b2b-08fd-4dc0-9de7-3c6ece09faf9.html) |
| 689f7782-ef2c-4270-a6d0-7664869076bd | [Configure Microsoft Defender CSPM to be enabled](https://www.azadvertizer.net/azpolicyadvertizer/689f7782-ef2c-4270-a6d0-7664869076bd.html) |

```bash
 
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

TEMP_POLICYSET_DEFINITION_NAME="custom-initiative-security-for-depenv"

az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV2}/providers/Microsoft.Authorization/policySetDefinitions/${TEMP_POLICYSET_DEFINITION_NAME}?api-version=2021-06-01" --body @- <<EOF
{
  "properties": {
    "displayName": "Security Policies for Sandbox",
    "description": "Modify/DINE policies fod sandbox subscriptions",
    "metadata": {
        "category": "Custom Initiative - Security"
    },
    "policyDefinitions": [
      {
        "policyDefinitionReferenceId": "ConfigureAzureDefenderforAzureSQLdatabasetobeenabled",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/b99b73e7-074b-4089-9395-b7236f094491",
        "parameters": {
          "effect": {
            "value": "DeployIfNotExists"
          }
        },
        "groupNames": []
      },
      {
        "policyDefinitionReferenceId": "ConfigureAzureDefenderforAppServicetobeenabled",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/b40e7bcd-a1e5-47fe-b9cf-2f534d0bfb7d",
        "parameters": {
          "effect": {
            "value": "DeployIfNotExists"
          }
        },
        "groupNames": []
      },
      {
        "policyDefinitionReferenceId": "ConfigureAzureDefenderforKeyVaultstobeenabled",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/1f725891-01c0-420a-9059-4fa46cb770b7",
        "parameters": {
          "effect": {
            "value": "DeployIfNotExists"
          }
        },
        "groupNames": []
      },
      {
        "policyDefinitionReferenceId": "ConfigureAzureDefenderforResourceManagertobeenabled",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/b7021b2b-08fd-4dc0-9de7-3c6ece09faf9",
        "parameters": {
          "effect": {
            "value": "DeployIfNotExists"
          }
        },
        "groupNames": []
      },
      {
        "policyDefinitionReferenceId": "ConfigureMicrosoftDefenderCSPMtobeenabled",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/689f7782-ef2c-4270-a6d0-7664869076bd",
        "parameters": {
          "effect": {
            "value": "DeployIfNotExists"
          }
        },
        "groupNames": []
      }
    ]
  }
}
EOF

# イニシアティブの割り当て ※ 割り当て名は MG に対しては最大 24 文字
az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV2}/providers/Microsoft.Authorization/policyAssignments/SecurityForSandbox?api-version=2022-06-01" --body @- <<EOF
{
  "location": "${LOCATION_NAMES[0]}",
  "properties": {
    "displayName": "Security for Sandbox",
    "scope": "/subscriptions/${SUBSCRIPTION_ID_DEV2}",
    "notScopes": [],
    "policyDefinitionId": "/subscriptions/${SUBSCRIPTION_ID_DEV2}/providers/Microsoft.Authorization/policySetDefinitions/${TEMP_POLICYSET_DEFINITION_NAME}",
    "enforcementMode": "Default",
    "parameters": {},
    "nonComplianceMessages": [],
    "resourceSelectors": [],
    "overrides": []
  },
  "identity": {
    "type": "SystemAssigned"
  }
}
EOF

```
