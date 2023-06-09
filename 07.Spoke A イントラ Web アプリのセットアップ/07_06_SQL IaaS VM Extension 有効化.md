# SQL IaaS VM Extension 有効化

```bash
 
# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
az account set -s "${SUBSCRIPTION_ID_SPOKE_A}"
 
# AMA に加えて、以下 3 つの Extension を入れる
# ※ SQLIaaS の VA/ATP 機能は AMA に対応しているが、ベストプラクティスアセスメント機能が MMA にしか対応していない。これを使いたい場合には別途 MMA を追加する。
# SQLIaaS : SqlIaasExtension (Microsoft.SqlServer.Management.SqlIaaSAgent)
# SQLVA : VulnerabilityAssessment.Windows (Microsoft.Azure.AzureDefenderForSQL.VulnerabilityAssessment.Windows)
# SQLATP : AdvancedThreatProtection.Windows (Microsoft.Azure.AzureDefenderForSQL.AdvancedThreatProtection.Windows)
 
# ① プロバイダと自動登録機能の有効化
# 下記により SQL IaaS VM 拡張機能の有効化と自動インストールが有効化される
 
az provider register --namespace Microsoft.SqlVirtualMachine
az feature register --name BulkRegistration --namespace Microsoft.SqlVirtualMachine
while [ $(az feature show --namespace Microsoft.SqlVirtualMachine --name BulkRegistration --query properties.state -o tsv) != "Registered" ]
do
  echo "$(az feature show --namespace Microsoft.SqlVirtualMachine --name BulkRegistration --query properties.state -o tsv) BulkRegistration..."
  sleep 10
done
 
# ② Defender for SQL machines の有効化
# LAWへのソリューションを追加し、SQLVA, SQLATP のサーバ側機能を有効化する
 
# ③ SQL IaaSエージェントのインストール
# 自動プロビジョングは時間がかかるので、下記で自力インストールしてしまってもよい
# OS と SQL の組み合わせによりモードに制限があるため注意する
# Full モード → Windows のみ利用可能
# NoAgent モード → SQL 2012 用
# 基本的に Windows なら Full モード、Linux なら LightWeight モードでインストール
# MDfSQLVM は Full モードでないと使えない
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
TEMP_VM_NAME=vm-db-${TEMP_LOCATION_PREFIX}
TEMP_RG_NAME="rg-spokea-${TEMP_LOCATION_PREFIX}"
 
# Register Enterprise or Standard self-installed VM in full mode
# az sql vm create --name <vm_name> --resource-group <resource_group_name> --location <vm_location> --license-type <license_type> --sql-mgmt-type Full
# [--image-sku {Developer, Enterprise, Express, Standard, Web}]
# [--license-type {AHUB, DR, PAYG}]
# [--sql-mgmt-type {Full, LightWeight, NoAgent}]
# Linux の場合は LightWeight のみ可
 
az sql vm create --name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --location $TEMP_LOCATION_NAME --license-type PAYG --sql-mgmt-type Full --image-sku Developer
 
# 上記の作業により、SqlIaasExtension がインストールされる
 
# ④ SQLVA, SQLATP のプロビジョニング
az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "VulnerabilityAssessment.Windows" --publisher "Microsoft.Azure.AzureDefenderForSQL"
az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "AdvancedThreatProtection.Windows" --publisher "Microsoft.Azure.AzureDefenderForSQL"
 
# ⑤ MMAのセットアップ （※ Windows のみ）
# ベストプラクティスアセスメントツールは現状 AMA 非対応
# このため MMA の追加セットアップが必要
# ※ *.blob を解放しなくても動作するようだが、公式には *.blob 解放が必要なため、商用環境では AMA 対応を待つことを推奨
 
# LAW の共有キーを取得（一時的にアカウントとサブスクリプションを切り替えてキーを取得）
az account set -s "${SUBSCRIPTION_ID_MGMT}"
  TEMP_LAW_NAME="law-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_LAW_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/${TEMP_LAW_NAME}"
  TEMP_RG_NAME="rg-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_LAW_KEY=$(az monitor log-analytics workspace get-shared-keys --name ${TEMP_LAW_NAME} --resource-group ${TEMP_RG_NAME} --query primarySharedKey -o tsv)
  TEMP_LAW_CUSTOMER_ID=$(az monitor log-analytics workspace show --workspace-name $TEMP_LAW_NAME --resource-group $TEMP_RG_NAME --query customerId -o tsv)
 
# 元に戻して作業を続行
az account set -s "${SUBSCRIPTION_ID_SPOKE_A}"
 
TEMP_VM_NAME=vm-db-${TEMP_LOCATION_PREFIX}
TEMP_RG_NAME="rg-spokea-${TEMP_LOCATION_PREFIX}"
 
# コマンドラインから展開するとうまくいかないので ARM テンプレートで展開
# az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "MicrosoftMonitoringAgent" --publisher "Microsoft.EnterpriseCloud.Monitoring" --settings '{"workspaceId":"${TEMP_LAW_WS_ID}"}' --protected-settings '{"workspaceKey":"${TEMP_LAW_KEY}"}'
 
cat <<EOF > tmp.json
{
  "\$schema": " https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Compute/virtualMachines/extensions",
      "apiVersion": "2021-11-01",
      "name": "${TEMP_VM_NAME}/Microsoft.Insights.LogAnalyticsAgent",
      "location": "${TEMP_LOCATION_NAME}",
      "properties": {
        "publisher": "Microsoft.EnterpriseCloud.Monitoring",
        "type": "MicrosoftMonitoringAgent",
        "typeHandlerVersion": "1.0",
        "autoUpgradeMinorVersion": true,
        "settings": {
          "workspaceId": "${TEMP_LAW_CUSTOMER_ID}"
        },
        "protectedSettings": {
          "workspaceKey": "${TEMP_LAW_KEY}"
        }
      }
    }
  ]
}
EOF
az deployment group create --name "${TEMP_VM_NAME}_Microsoft.Insights.LogAnalyticsAgent"  --resource-group "${TEMP_RG_NAME}" --template-file tmp.json
 
done
 
 
# ⑥ ベストプラクティスアセスメントの実行 （※ Windows のみ）
# ※ Full モードの SQL VM に対してしかスキャンできない (= Linux SQL IaaS VM は現状スキャンできない)
# https://learn.microsoft.com/en-us/rest/api/sqlvm/2022-07-01-preview/sql-virtual-machines/start-assessment?tabs=HTTP
 
# ベストプラクティスアセスメントの実行には Microsoft.SqlVirtualMachine/sqlVirtualMachines/startAssessment/action 権限が必要
# この権限はガバナンスチームは持っていないため、業務システム A のチームメンバーから実施
 
# 業務システム A チーム／② 平常作業の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spokea_ops@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
TEMP_VM_NAME=vm-db-${TEMP_LOCATION_PREFIX}
TEMP_RG_NAME="rg-spokea-${TEMP_LOCATION_PREFIX}"
 
az rest --method post --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID_SPOKE_A}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.SqlVirtualMachine/sqlVirtualMachines/${TEMP_VM_NAME}/startAssessment?api-version=2022-07-01-preview"
 
done
```
