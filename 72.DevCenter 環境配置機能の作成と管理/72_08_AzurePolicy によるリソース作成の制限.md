# Azure Policy によるリソース作成の制限

前項ではリソースプロバイダによるリソース作成の制約・制限について解説しましたが、Azure Policy を利用しても似たようなリソース作成の制限を行うことができます。

- リソースプロバイダは機能そのものの有効化／無効化を設定するもの
- Azure Policy は、リソースプロバイダに対する操作内容を制限するもの

後者については、操作パラメータ内容の制限・チェックなども可能なため、より細かい制限が可能です。ここでは具体例として、2 つの例を示します。

- 利用可能なリソース種別の制限
- 作成可能な VM サイズの制限

制限をかける場合、カスタムポリシーを開発する前に、利用できる Azure Policy がないかを探してみてください。類似のものが見つかれば、カスタムポリシーを開発する場合でも多少ラクができます。

- [MS Learn](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies#general)
- [AzAdvertizer](https://www.azadvertizer.net/index.html)

## 利用可能なリソース種別の制限

以下の Azure Policy を利用することで、作成するリソース種別を制限できます。

- Allowed resource types
  - [a08ec900-254a-4555-9bf5-e42af04b5c5c](https://www.azadvertizer.net/azpolicyadvertizer/a08ec900-254a-4555-9bf5-e42af04b5c5c.html)
  - パラメータ listOfResourceTypesAllowed により、作成を許可するリソースを指定できる
- Not allowed resource types
  - [6c112d4e-5bc7-47ae-a041-ea2d9dccd749](https://www.azadvertizer.net/azpolicyadvertizer/6c112d4e-5bc7-47ae-a041-ea2d9dccd749.html)
  - パラメータ listOfResourceTypesNotAllowed により、作成を許可するリソースを指定できる
  - パラメータ effect により動作モードを指定する

（参考）実際の利用では、Allowed resource types よりも、**Not allowed resource types の利用**をオススメします。これは、Allowed resource type を利用した場合、列挙しなければならないリソース種別が非常に多くなりすぎてしまうためです。Not allowed resoruce types を利用した場合、新サービスが登場した際に漏れる可能性がありますが、リソースプロバイダ登録の有無でも機能制限がかかるため、問題になりにくいです。

## 作成可能な VM サイズの制限

以下の Azure Policy を利用することで、作成する VM サイズを制限できます。

- Allowed virtual machine size SKUs
  - [cccc23c7-8427-4f53-ad12-b6a63eb452b3](https://www.azadvertizer.net/azpolicyadvertizer/cccc23c7-8427-4f53-ad12-b6a63eb452b3.html)
  - パラメータ listOfAllowedSKUs により、作成を許可する VM サイズを指定できる

## ポリシーイニシアティブの作成と割り当て

これらのポリシーは、取り扱いを容易にするため、いったんイニシアティブに束ねてから適用します。

```bash

# VM SKU 一覧
# az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID_NGNT}/providers/Microsoft.Compute/locations/eastus/vmSizes?api-version=2023-07-01" --query value[].name

if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
TEMP_POLICYSET_DEFINITION_NAME="custom-initiative-restriction-for-depenv"

az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV2}/providers/Microsoft.Authorization/policySetDefinitions/${TEMP_POLICYSET_DEFINITION_NAME}?api-version=2021-06-01" --body @- <<EOF
{
  "properties": {
    "displayName": "Restriction Policies for Sandbox",
    "description": "Deny Policies for sandbox subcrtipions",
    "metadata": {
      "category": "Custom Initiative - Guardrail"
    },
    "policyDefinitions": [
      {
        "policyDefinitionReferenceId": "NotAllowedResourceTypes_Deny",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/6c112d4e-5bc7-47ae-a041-ea2d9dccd749",
        "parameters": {
          "effect": {
            "value": "Deny"
          },
          "listOfResourceTypesNotAllowed": {
            "value": [
              "Microsoft.CognitiveServices/accounts"
            ]
          }
        },
        "groupNames": []
      },
      {
        "policyDefinitionReferenceId": "AllowedVirtualMachineSizeSKUs_Deny",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/cccc23c7-8427-4f53-ad12-b6a63eb452b3",
        "parameters": {
          "listOfAllowedSKUs": {
            "value": [
              "Standard_D2s_v5",
              "Standard_D4s_v5",
              "Standard_D8s_v5",
              "Standard_D16s_v5"
            ]
          }
        },
        "groupNames": []
      }
    ]
  }
}
EOF

# (参考) 許可するリソース種別の一覧を使う場合
      # {
      #   "policyDefinitionReferenceId": "AllowedResourceTypes_Deny",
      #   "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/a08ec900-254a-4555-9bf5-e42af04b5c5c",
      #   "parameters": {
      #     "listOfResourceTypesAllowed": {
      #       "value": [
      #         "Microsoft.App/containerApps",
      #         "Microsoft.App/managedEnvironments",
      #         "Microsoft.ContainerRegistry/registries",
      #         "Microsoft.Sql/servers",
      #         "Microsoft.Sql/servers/databases",
      #         "Microsoft.Security/automations"
      #       ]
      #     }
      #   },
      #   "groupNames": []
      # },


az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV2}/providers/Microsoft.Authorization/policyAssignments/RestrictionForSandbox?api-version=2022-06-01" --body @- <<EOF
{
  "properties": {
    "displayName": "Restriction for Sandbox",
    "scope": "/subscriptions/${SUBSCRIPTION_ID_DEV2}",
    "notScopes": [],
    "policyDefinitionId": "/subscriptions/${SUBSCRIPTION_ID_DEV2}/providers/Microsoft.Authorization/policySetDefinitions/${TEMP_POLICYSET_DEFINITION_NAME}",
    "enforcementMode": "Default",
    "parameters": {},
    "nonComplianceMessages": [],
    "resourceSelectors": [],
    "overrides": []
  }
}
EOF

```
