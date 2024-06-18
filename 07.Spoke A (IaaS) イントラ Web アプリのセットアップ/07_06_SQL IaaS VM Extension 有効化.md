# SQL IaaS VM Extension 有効化

Azure 環境で SQL Server を利用したい場合には、通常は PaaS 版 SQL Server である SQL Database サービスをご利用いただくのが便利です（詳細は業務システム B にて示します）。しかし SQL Database は SQL Server のフルセット機能が利用できるわけではないため、既存システムの Lift & Shift のような場合には、本デモのように、仮想マシンを利用して SQL Server を構築する場合もあります。

このような場合に備えて、Azure では SQL IaaS VM Extension と呼ばれる VM 拡張機能が用意されています。主には以下のような機能が提供されます。特にライセンス管理やコンプライアンス準拠の観点から、IaaS VM で SQL Server をご利用いただいている場合には必ずインストールしていただくようにしてください。

- 管理性の向上
  - Azure ポータルでの SQL VM 管理画面の提供
  - バックアップ・パッチ適用のスケジューリング
  - tempdb 構成の変更、ディスク使用率の表示
  - 可用性グループ構成の容易化
- セキュリティ機能の提供
  - Azure Key Vault 統合機能の提供
  - Defender for SQL Server machines の提供（SQLVA, SQLATP）（※ 有償）
- ライセンス管理・コンプライアンス準拠
  - ライセンスモデル・エディションの変更
  - AHUB ライセンス特典の利用
  - DR 用ライセンスの利用
- アドバイザリ機能の提供
  - SQL ベストプラクティスアセスメント機能の提供

![picture 1](./images/1c1c3690089776604b6ed0593e9734398a7298a7ecfd3f1699ea3674e8f73407.png)  

具体的なインストールスクリプトや使い方は以下の通りです

```bash
 
# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
az account set -s "${SUBSCRIPTION_ID_SPOKE_A}"
 
# AMA に加えて、以下の Extension を入れる
# SQLATP : AdvancedThreatProtection.Windows (Microsoft.Azure.AzureDefenderForSQL.AdvancedThreatProtection.Windows) ※ v2.0 以上

# プロバイダと自動登録機能の有効化
# 下記により SQL IaaS VM 拡張機能の有効化と自動インストールが有効化される
 
az provider register --namespace Microsoft.SqlVirtualMachine
az feature register --name BulkRegistration --namespace Microsoft.SqlVirtualMachine
while [ $(az feature show --namespace Microsoft.SqlVirtualMachine --name BulkRegistration --query properties.state -o tsv) != "Registered" ]
do
  echo "$(az feature show --namespace Microsoft.SqlVirtualMachine --name BulkRegistration --query properties.state -o tsv) BulkRegistration..."
  sleep 10
done
 
# SQL IaaSエージェントのインストール
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
# SQLATP のプロビジョニング
az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "AdvancedThreatProtection.Windows" --publisher "Microsoft.Azure.AzureDefenderForSQL" --version "2.0"
 
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

done # TEMP_LOCATION

```
