# Azure Policy (Remediation)

ポリシー違反項目に対して、リソース側を正しく是正します。

- 本サンプルでは構築スクリプト側をすでに修正してあるため、リソース作成後に是正すべき構成違反（ポリシー違反項目）はありません。
- 一方、以下のようなポリシー違反が発生する場合があります。これらに関しては、適切な対処が必要です。
  - VM にセキュリティパッチが適用されていない
    - System updates should be installed on your machines (powered by Azure Update Manager)
  - ACR 内のイメージや AKS で実行されているコンテナイメージの中に、脆弱性のあるライブラリが存在する
    - Azure running container images should have vulnerabilities resolved
    - [Preview] Container images in Azure registry should have vulnerability findings resolved
    - Azure registry container images should have vulnerabilities resolved

## VM へのセキュリティパッチ適用

- 以下のスクリプトを実行し、VM へセキュリティパッチを適用します。
- 下記ではすべてのサブスクリプションを対象に、セキュリティパッチのアセスメントとインストールを実行します。実際の運用ではスクリプトを適宜修正して利用してください。

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

TEMP_TARGET_SUBSCRIPTION_IDS=$SUBSCRIPTION_IDS

# パッチの評価
for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
az account set -s "${TEMP_SUBSCRIPTION_ID}"
for TEMP_RG_NAME in $(az group list --query "[].name" -o tsv); do
for TEMP_VM_NAME in $(az vm list --resource-group ${TEMP_RG_NAME} --query "[].name" -o tsv); do
 
# https://learn.microsoft.com/ja-jp/azure/update-manager/manage-vms-programmatically
TEMP_VM_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}"
echo "Assess Pathes on ${TEMP_SUBSCRIPTION_ID} ${TEMP_RG_NAME} ${TEMP_VM_NAME}"
az rest --method POST --uri "${TEMP_VM_ID}/assessPatches?api-version=2020-12-01"

done #TEMP_VM_NAME
done #TEMP_RG_NAME
done #TEMP_SUBSCRIPTION_ID

# パッチのインストール
for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
az account set -s "${TEMP_SUBSCRIPTION_ID}"
for TEMP_RG_NAME in $(az group list --query "[].name" -o tsv); do
for TEMP_VM_NAME in $(az vm list --resource-group ${TEMP_RG_NAME} --query "[].name" -o tsv); do
 
# https://learn.microsoft.com/ja-jp/azure/update-manager/manage-vms-programmatically
TEMP_VM_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}"

TEMP="OperationNotAllowed"
while [[ ${TEMP} =~ "OperationNotAllowed" ]]
do
  echo "Trying to install Pathes on ${TEMP_SUBSCRIPTION_ID} ${TEMP_RG_NAME} ${TEMP_VM_NAME}"
  TEMP=$(az rest --method POST --uri "${TEMP_VM_ID}/installPatches?api-version=2020-12-01" 2>&1 --body @- <<EOF
{
  "maximumDuration": "PT120M",
  "rebootSetting": "IfRequired",
  "windowsParameters": {
    "classificationsToInclude": [
      "Security",
      "UpdateRollup",
      "FeaturePack",
      "ServicePack"
    ]
  }
}
EOF
)
  echo $TEMP
  sleep 10
done

done #TEMP_VM_NAME
done #TEMP_RG_NAME
done #TEMP_SUBSCRIPTION_ID

```

## イメージの再作成

コンテナイメージに脆弱性がある場合には、以下を行います。

- ソースコードの修正
  - まずは開発環境において、ソースコード（ライブラリ参照など）を適宜修正します。
- 脆弱性有無の確認
  - 開発環境で、作成したイメージに脆弱性が残っていないかを確認します。
  - 確認には Trivy を利用するのが便利です。
    - [インストール方法](https://aquasecurity.github.io/trivy/v0.53/getting-started/installation/)
    - sudo trivy image app (app という名前のイメージのセキュリティスキャン を実施)
- ACR へのプッシュ
  - [62_04_コンテナアプリのビルド](../62.Spoke%20F%20(CaaS)%20コンテナビルドと%20Web-DB%20アプリの配置/62_04_コンテナアプリのビルド.md)を再度行い、ソースコードの再取得やイメージの再作成、ACR へのプッシュを実施します。
- コンテナ実行環境の最新化
  - コンテナ環境をリフレッシュし、最新のイメージを取得させます。
  - Azure Container Apps の場合には、
- ACR からの古いイメージの削除
  - ACR のプライベートエンドポイントに到達できるマシンから、ACR 上に存在する古いイメージを削除します。
