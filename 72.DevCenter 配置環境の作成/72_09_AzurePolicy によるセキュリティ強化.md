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

これらはサンドボックスサブスクリプションを管理する管理グループに対して仕掛けると効果的です。

```bash
 
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi

TEMP_MG_SANDBOX_ID=$(az account management-group list --query "[?name=='sandbox'].id" -o tsv)

# カスタムイニシアティブの作成
cat > temp.json << EOF
{
  "\$schema": " https://schema.management.azure.com/schemas/2018-05-01/subscriptionDeploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Authorization/policySetDefinitions",
      "name": "custom-initiative-secure-sandbox",
      "apiVersion": "2019-09-01",
      "properties": {
        "displayName": "Security for Sandbox Policies",
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
  ]
}
EOF

TEMP=(${TEMP_MG_SANDBOX_ID//\// })
az deployment mg create --location ${LOCATION_NAMES[0]} --name "custom-initiatives-sandbox-policies" --template-file temp.json --management-group-id "${TEMP[3]}"

# イニシアティブの割り当て ※ 割り当て名は MG に対しては最大 24 文字
cat > temp.json << EOF
{
  "\$schema": " https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Authorization/policyAssignments",
      "name": "custom-sandbox-security",
      "apiVersion": "2019-09-01",
      "location": "${LOCATION_NAMES[0]}",
      "properties": {
        "displayName": "Secure Sandbox Subscriptions",
        "description": "サンドボックスサブスクリプションのセキュリティを強化します。",
        "scope": "/providers/Microsoft.Management/managementGroups/sandbox",
        "policyDefinitionId": "${TEMP_MG_SANDBOX_ID}/providers/Microsoft.Authorization/policySetDefinitions/custom-initiative-secure-sandbox"
      },
      "identity": {
        "type": "SystemAssigned"
      }
    }
  ]
}
EOF

az deployment mg create --location ${LOCATION_NAMES[0]} --name "custom-sandbox-check" --template-file temp.json --management-group-id "${TEMP[3]}"

```
