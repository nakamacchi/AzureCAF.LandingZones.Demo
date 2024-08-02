# k8s 用 Azure Policy の調整

k8s 用 Azure Policy Add-on を用いて構成チェックを行いたい場合、いくつかの項目については自環境に合わせたパラメータ設定が必要になります([docs](https://learn.microsoft.com/en-us/azure/defender-for-cloud/kubernetes-workload-protections#view-and-configure-the-bundle-of-recommendations), [詳細](https://qiita.com/hisnakad/items/79d5c00052ac6be5ac71))。

## 対応方針について

k8s 用ポリシーに関しては、ポリシーごとに対応方針を分ける必要があります。以下の 4 つに大別して考えます。

- ① 必ず設定すべきもので全体に対して共通のパラメータで適用できるものは、ルートで設定する
- ② 必ず設定すべきだが個別にパラメータを変える必要があるものは、テナントルートで disabled（または緩い設定）にして、個々の SubScription でカスタムポリシーにより設定する
- ③ 必ず設定すべきだが当該アプリの都合で満たすことができないものは、Exemption 扱いにする（例：k8s dashboard にリソースリミット制限が設定されていない）
- ④ 自社の場合はポリシーとして厳しすぎるため、適用を見送るものは、テナントルートで disabled 扱いにする

注意すべき点として、「全社的（テナントルート）では禁止されているが、特定のサブスクリプションやリソースに対してだけポリシーを緩める」（親側は厳しいが子側で個別に緩める）ということはできません。このようなリソースは、③の Exemption として扱います。

## 具体的な設計例

今回、④に相当するものはないものとします。そのうえで、②・③に該当するために調整が必要なものは以下の通りです。

### ② 必ず設定すべきだが個別にパラメータを変える必要があるもの

今回の場合、特にパラメータ設定が必要な項目は以下の 2 つです。

| MDfC Recommendation | MDfC ID | Azure Policy | Policy ID |
| --- | --- | --- | --- |
| Container images should be deployed from trusted registries only | 8d244d29-fa00-4332-b935-c3a51d525417 | Kubernetes cluster containers should only use allowed images | febd0533-8e55-448f-b837-bd0e06f16469 |
| Services should listen on allowed ports only | add45209-73f6-4fa5-a5a5-74a451b07fbe | Kubernetes cluster services should listen only on allowed ports | 233a2a17-77ca-4fb1-9b6b-69223d272a44 |

この 2 つについては、テナントルートでの設定を大きく緩める、もしくは無効にし、Spoke E サブスクリプションに対してのみ個別設定を適用します。（※ この設定はすでに[こちら](../11.Azure%20Policy%20ガバナンスと%20MDfC%20セキュリティ/11_12_適用するポリシーの見直し（ポリシーの無効化）.md)で設定済みです。）

本アプリの場合、それぞれに対して設定すべきパラメータは以下の通りです。

- Container images should be deployed from trusted registries only
  - allowedContainerImagesRegex = "(^[^\/]+\.azurecr\.io\/.+$)|(^kubernetesui\/.+$)"
  - ACR のイメージに加え、k8s dashboard を利用する場合は、以下の 2 つのイメージを使います。
    - kubernetesui/metrics-scraper:v1.0.7
    - kubernetesui/dashboard:v2.5.0
- Services should listen on allowed ports only
  - allowedPortsList = \["8000","443","80"\]
  - k8s dashboard を使う場合はポート 8000 を利用します。

### ③ 必ず設定すべきだが当該アプリの都合で満たすことができないもの

今回の場合、いくつのか理由により Azure Policy の要件を満たせません。具体的には以下の通りです。

まず、コンテナ環境の更なるセキュリティ強化のため、以下のオプション指定が推奨されていますが、ASP.NET Core Blazor アプリでの適用は大変です。

```yaml
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: true
```

このため、以下のポリシーについては無効化します。

| MDfC Recommendation | MDfC ID | Azure Policy | Policy ID |
| --- | --- | --- | --- |
| Running containers as root user should be avoided | 9b795646-9130-41a4-90b7-df9eae2437c8 | Kubernetes cluster pods and containers should only run with approved user and group IDs | f06ddb64-5fa3-4b77-b166-acb36f7f6042 |
| Immutable (read-only) root filesystem should be enforced for containers | 27d6f0e9-b4d5-468b-ae7e-03d5473fd864 | Kubernetes cluster containers should run with a read only root file system | df49d893-a74c-421d-bc95-c663042e5b80 |

また k8s dashboard では以下に違反する設定が含まれています。このため、これらのポリシーに対しては除外設定を追加します。

| MDfC Recommendation | MDfC ID | Azure Policy | Policy ID |
| --- | --- | --- | --- |
| Container CPU and memory limits should be enforced | 405c9ae6-49f9-46c4-8873-a86690f27818 | Kubernetes cluster containers CPU and memory resource limits should not exceed the specified limits | e345eecc-fa47-480f-9e88-67dcc122b164 |
| Kubernetes clusters should disable automounting API credentials| | 32060ac3-f17f-4848-db8e-e7cf2c9a53eb | Kubernetes clusters should disable automounting API credentials | 423dd1ba-798e-40e4-9c4d-b6902674b423 |

これらのポリシーは、Step 55_05_AzurePolicy(Exemption-Waiver) にて除外適用します。

```bash

if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)

# カスタムイニシアティブの作成
az rest --method PUT --uri "${TEMP_MG_TRG_ID}/providers/Microsoft.Authorization/policySetDefinitions/custom-initiative-spoke-e?api-version=2023-04-01" --body @- <<EOF
{
    "properties": {
        "displayName": "Check Policies for Spoke E",
        "description": "Audit/AINE policies for Spoke E",
        "metadata": {
            "category": "Custom Initiative - Check"
        },
        "policyDefinitions": [
            {
                "policyDefinitionReferenceId": "ensureAllowedContainerImagesInKubernetesCluster",
                "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/febd0533-8e55-448f-b837-bd0e06f16469",
                "parameters": {
                    "effect": {
                        "value": "Audit"
                    },
                    "allowedContainerImagesRegex": {
                        "value": "(^[^\/]+\.azurecr\.io\/.+$)|(^kubernetesui\/.+$)"
                    },
                    "excludedNamespaces": {
                        "value": [
                            "kube-system",
                            "gatekeeper-system",
                            "azure-arc",
                            "azure-extensions-usage-system"
                        ]
                    },
                    "labelSelector": {
                        "value": {}
                    }
                },
                "groupNames": [
                ]
            },
            {
                "policyDefinitionReferenceId": "allowedServicePortsInKubernetesCluster",
                "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/233a2a17-77ca-4fb1-9b6b-69223d272a44",
                "parameters": {
                    "effect": {
                        "value": "Audit"
                    },
                    "allowedServicePortsList": {
                        "value": [
                            "8000",
                            "443",
                            "80"
                        ]
                    },
                    "excludedNamespaces": {
                        "value": [
                            "kube-system",
                            "gatekeeper-system",
                            "azure-arc",
                            "azure-extensions-usage-system"
                        ]
                    },
                    "labelSelector": {
                        "value": {}
                    }
                },
                "groupNames": [
                ]
            }
        ]
    }
}
EOF

# イニシアティブの割り当て ※ 割り当て名は MG に対しては最大 24 文字
TEMP_SCOPE="/subscriptions/${SUBSCRIPTION_ID_SPOKE_E}"
az rest --method PUT --uri "${TEMP_SCOPE}/providers/Microsoft.Authorization/policyAssignments/custom-check-spe?api-version=2023-04-01" --body @- <<EOF
{
    "properties": {
        "displayName": "Check for Spoke E AKS",
        "description": "Spoke E の AKS を Audit/AINE カスタムポリシーで検査します。",
        "scope": "${TEMP_SCOPE}",
        "policyDefinitionId": "${TEMP_MG_TRG_ID}/providers/Microsoft.Authorization/policySetDefinitions/custom-initiative-spoke-e"
    }
}
EOF

```

