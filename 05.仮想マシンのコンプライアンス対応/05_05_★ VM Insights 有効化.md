# ★ VM Insights 有効化

仮想マシンの通信やパフォーマンスを解析するためのツールである VM Insights を有効化します。

- 必要なエージェントである AMA と DA をインストールします。
- [以前のステップ](/04.%E7%AE%A1%E7%90%86%E5%9F%BA%E7%9B%A4%E3%81%AE%E6%A7%8B%E6%88%90%E8%A8%AD%E5%AE%9A/04_01_VMInsights%E7%94%A8%E3%81%AE%E4%BA%8B%E5%89%8D%E6%BA%96%E5%82%99.md)で作成した DCR を当該マシンに割り当てます。

スクリプト実行後、いくつかの処理を Azure ポータルなどから手作業で行う必要があります。

- DCE (Data Collection Endpoint)の割り当て
  - カスタムログなどを転送する場合に備えて、DCE の利用割り当てを行う必要があります。
  - ~~API は存在しており、ポータルからの設定でも API を叩いているようなのですが、うまく設定が反映されず。。。というわけで申し訳ありませんが、ここだけ暫定的に手作業での割り当てをお願いします。m(_ _)m　（作業方法はビデオを確認してください。）~~ → 修正済みです。スクリプトでの割り当てが可能になりました。
- Linux OS のカーネル変更
  - VM Insights の通信モニタリング機能は、残念ながら Linux の最新カーネルでは動作しないという問題があります。利用条件は[こちら](https://learn.microsoft.com/ja-jp/azure/azure-monitor/vm/vminsights-enable-overview)。
  - Ubuntu 20 などではカーネルをダウングレードすることにより機能の有効化が可能です。（ダウングレードを推奨するものでもないので、やってみたい場合には本スクリプトの後半に書かれている方法で有効化してみてください。）

```bash

# AMA インストールと DCR の割り当てによる VM Insights の有効化

for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
echo "Setting VM Insights... ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# DCE, DCR はリージョンごとの LAW に対して作成しているのでそれを利用
TEMP_DCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionEndpoints/dce-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_DCR_AMA_WIN_NAME="dcr-law-vdc-${TEMP_LOCATION_PREFIX}-ama-win"
TEMP_DCR_AMA_LINUX_NAME="dcr-law-vdc-${TEMP_LOCATION_PREFIX}-ama-linux"
TEMP_DCR_VMI_NAME="dcr-law-vdc-${TEMP_LOCATION_PREFIX}-vmi"
TEMP_DCR_AMA_WIN_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionRules/${TEMP_DCR_AMA_WIN_NAME}"
TEMP_DCR_AMA_LINUX_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionRules/${TEMP_DCR_AMA_LINUX_NAME}"
TEMP_DCR_VMI_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionRules/${TEMP_DCR_VMI_NAME}"

# 当該リージョンのリソースグループを拾って処理
for TEMP_RG_NAME in $(az group list --query "[?location == '${TEMP_LOCATION_NAME}' ].name" -o tsv); do

# エージェントのインストール (AMA, DA)
# 動作前提条件として Managed ID の有効化が必要（ここではシステム割り当てを利用）
# その他の動作前提条件 → https://docs.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-manage#prerequisites
# エージェントのインストールは比較的時間がかかるため、エージェントが入っている場合には処理をスキップするようにしています。

# Windows OS
for TEMP_VM_NAME in $(az vm list --resource-group $TEMP_RG_NAME --query [?osProfile.windowsConfiguration!=null].name -o tsv); do
echo "Installing agents on ${TEMP_VM_NAME}..."
if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='AzureMonitorWindowsAgent']" -o tsv)" ]; then
az vm identity assign --name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME}
az vm extension set --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name "AzureMonitorWindowsAgent" --publisher "Microsoft.Azure.Monitor" --enable-auto-upgrade true
fi
if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='DependencyAgentWindows']" -o tsv)" ]; then
az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --name "DependencyAgentWindows"  --publisher "Microsoft.Azure.Monitoring.DependencyAgent" --settings "{\"enableAMA\": \"true\"}"
fi
if [ -n "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='MicrosoftMonitoringAgent']" -o tsv)" ]; then
# MMA は不要なので削除 (基本的にインストールされていないはずだが)
az vm extension delete --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name MicrosoftMonitoringAgent
fi
done # TEMP_VM_NAME
# Linux OS
for TEMP_VM_NAME in $(az vm list --resource-group $TEMP_RG_NAME --query [?osProfile.linuxConfiguration!=null].name -o tsv); do
echo "Installing agents on ${TEMP_VM_NAME}..."
if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='AzureMonitorLinuxAgent']" -o tsv)" ]; then
az vm identity assign --name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME}
az vm extension set --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name "AzureMonitorLinuxAgent" --publisher "Microsoft.Azure.Monitor" --enable-auto-upgrade true
fi
if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='DependencyAgentLinux']" -o tsv)" ]; then
az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --name "DependencyAgentLinux"  --publisher "Microsoft.Azure.Monitoring.DependencyAgent" --settings "{\"enableAMA\": \"true\"}"
fi
if [ -n "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='OmsAgentForLinux']" -o tsv)" ]; then
# OMA は不要なので削除 (基本的にインストールされていないはずだが)
az vm extension delete --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name OmsAgentForLinux
fi
done # TEMP_VM_NAME

# DCR/DCE 割り当て
# DCR の割り当てには、① 対象 VM に対する Microsoft.Insights/dataCollectionRuleAssociations/write と、② DCR に対する Microsoft.Insights/dataCollectionRules/read の権限が必要。
# DCE の割り当てには、DCE に対する Microsoft.Insights/dataCollectionEndpoints/write の権限が必要。

# DCR 割り当て
# Windows マシンへの適用
for TEMP_VM_ID in $(az vm list --resource-group $TEMP_RG_NAME --query [?osProfile.windowsConfiguration!=null].id -o tsv); do
echo "Setting DCR on ${TEMP_VM_ID}..."
az monitor data-collection rule association create --name "${TEMP_DCR_AMA_WIN_NAME}-association" --rule-id ${TEMP_DCR_AMA_WIN_ID} --resource ${TEMP_VM_ID}
az monitor data-collection rule association create --name "${TEMP_DCR_VMI_NAME}-association" --rule-id ${TEMP_DCR_VMI_ID} --resource ${TEMP_VM_ID}
az rest --method put --url "${TEMP_VM_ID}/providers/microsoft.insights/dataCollectionRuleAssociations/configurationAccessEndpoint?api-version=2021-04-01" --body "{ \"properties\": { \"dataCollectionEndpointId\": \"${TEMP_DCE_ID}\" } }"
done
# Linux マシンへの適用
for TEMP_VM_ID in $(az vm list --resource-group $TEMP_RG_NAME --query [?osProfile.linuxConfiguration!=null].id -o tsv); do
echo "Setting DCR on ${TEMP_VM_ID}..."
az monitor data-collection rule association create --name "${TEMP_DCR_AMA_LINUX_NAME}-association" --rule-id ${TEMP_DCR_AMA_LINUX_ID} --resource ${TEMP_VM_ID}
az monitor data-collection rule association create --name "${TEMP_DCR_VMI_NAME}-association" --rule-id ${TEMP_DCR_VMI_ID} --resource ${TEMP_VM_ID}
done

# DCE 割り当て（VM 単位に行えばすべての DCR に対して有効になる）
# → うまく動作しない、GUI から実行するとうまくいくのでそちらで対応
for TEMP_VM_ID in $(az vm list --resource-group $TEMP_RG_NAME --query [].id -o tsv); do
echo "Setting DCE on ${TEMP_VM_ID}..."
az rest --method put --url "${TEMP_VM_ID}/providers/microsoft.insights/dataCollectionRuleAssociations/configurationAccessEndpoint?api-version=2021-04-01" --body "{ \"properties\": { \"dataCollectionEndpointId\": \"${TEMP_DCE_ID}\" } }"
done

done # TEMP_RG_NAME
done # TEMP_LOCATION_NAME
done # TEMP_SUBSCRIPTION_ID

# （参考）Linux OS における VM Insights（サービスマップ）の有効化について
# https://learn.microsoft.com/ja-jp/azure/azure-monitor/vm/vminsights-enable-overview#linux-considerations
# VM Insights は Ubuntu 22 非対応（Ubuntu 20 まで）
# Ubuntu 20 の場合でも、そのままだと Dependency Agent は 以下のエラーでインストールされない → Missing kernel symbol: printk エラー、これは kernel Version が 5.8 までしかサポートされていないため。以下の方法で Kernel のダウングレードを行う
# https://docs.microsoft.com/en-us/azure/azure-monitor/vm/vminsights-dependency-agent-maintenance#dependency-agent-linux-support
# sudo uname -a
# カーネルのダウングレード方法は以下を参照
# https://qiita.com/ego/items/36e9baccc80097950195
# sudo apt-cache search linux-image
# 上記から signed の azure イメージ (5.8 ベース) の最新を探す
# sudo apt-get install linux-image-5.8.0-1043-azure --yes
# https://level69.net/archives/31142
# vi コマンドで grub ファイルを編集
# sudo vi /etc/default/grub
# ※ esc > i で編集モードに入り以下を修正
# GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.8.0-1043-azure"
# ※ esc > :wq で保存と終了
# sudo update-grub
# sudo reboot
# リブート後、カーネルが変更されていることを確認
# sudo uname -a
# この設定が行われていない VM に対しては printf がないとしてエラーが出る。
 
```
