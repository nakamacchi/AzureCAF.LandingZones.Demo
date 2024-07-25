# 是正処理 (Remediation) : よくある是正処理

是正可能なものは適宜修正を行う必要があります。ここでは、以下の是正処理を行います。

- フローログの有効化
  - NSG フローログを収集するようにシステムを構成します。
  - NSG フローログの収集にはストレージアカウントが必要ですが、作成したストレージアカウントに対しても ASB によるチェックがかかります。
- Windows VM への TLS 1.2 の強制
  - Windows マシン用の GC のルールの一つである、AuditSecureProtocol ルールの是正方法を示します。このルールは　TLS 1.2 以外のプロトコルが利用不可になっているかを確認するもので、プロトコルを無効化するためにはレジストリを設定します。
  - コマンドラインから実行する方法は[こちら](/99.Tips/99_03_Windows%20OS%20%E3%81%A7%E3%81%AE%20TLS%201.2%20%E5%BC%B7%E5%88%B6.md)に示していますが、VM にログインするのは面倒なため、runCommand API を使って処理します。
- セキュリティパッチの適用指示
  - 未適用のセキュリティパッチがある場合、MDE がこれを検出し、推奨事項（非準拠事項）として報告してきます。セキュリティパッチは速やかに適用するようにしてください。
  - セキュリティパッチの適用は、UMC (Update Management Center) と呼ばれる機能を使って行うことができます。仮想マシンに標準インストールされている VM ゲストエージェントの機能が利用されるため、特に追加のエージェントのインストールは不要です。

またリソース作成後、しばらくしてから[リソース診断ログの出力設定](/01.%E5%88%9D%E6%9C%9F%E7%92%B0%E5%A2%83%E3%82%BB%E3%83%83%E3%83%88%E3%82%A2%E3%83%83%E3%83%97/01_04_%E2%98%85%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E8%A8%BA%E6%96%AD%E3%83%AD%E3%82%B0%E5%87%BA%E5%8A%9B%E3%81%AE%E4%B8%80%E6%8B%AC%E8%A8%AD%E5%AE%9A.md)も行います。（追加作成したリソースに対してリソース診断ログ出力を設定します。）

```bash
 
#####################################
# フローログの有効化

if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_plat_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_plat_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
 
# FlowLog 用ストレージアカウントの作成 (リージョン別)
az account set -s "${SUBSCRIPTION_ID_MGMT}"
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
  TEMP_RG_NAME="rg-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_LAW_NAME="law-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_DCE_NAME="dce-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_FLOWLOG_STORAGE_NAME="stvdcfl${TEMP_LOCATION_PREFIX}${UNIQUE_SUFFIX}"
 
  az storage account create --name ${TEMP_FLOWLOG_STORAGE_NAME} --resource-group ${TEMP_RG_NAME} --location $TEMP_LOCATION_NAME --sku Standard_LRS --allow-blob-public-access false --public-network-access Disabled --default-action Deny --allow-shared-key-access false
done
 
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
 
# FlowLog 設定
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
  TEMP_RG_NAME="rg-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_LAW_NAME="law-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_FLOWLOG_STORAGE_NAME="stvdcfl${TEMP_LOCATION_PREFIX}${UNIQUE_SUFFIX}"
 
TEMP_LAW_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.operationalinsights/workspaces/${TEMP_LAW_NAME}"
TEMP_FLOWLOG_STORAGE_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Storage/storageAccounts/${TEMP_FLOWLOG_STORAGE_NAME}"
 
# 各リージョン内の NIC を上記ストレージと LAW に入れる
for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_IDS; do
 
az account set -s $TEMP_SUBSCRIPTION_ID
 
# NSG を検索して NSG フローログを有効化
for TEMP_NSG_ID in $(az resource list --query "[?type == 'Microsoft.Network/networkSecurityGroups' && location == '${TEMP_LOCATION_NAME}'].id" -o tsv); do
  echo $TEMP_NSG_ID
  TEMP=(${TEMP_NSG_ID//\// })
  TEMP_RG_NAME=${TEMP[3]}
  TEMP_NSG_NAME=${TEMP[7]}
  echo $TEMP_RG_NAME $TEMP_NSG_NAME
  az network watcher flow-log create --location ${TEMP_LOCATION_NAME} --name ${TEMP_NSG_NAME} --nsg ${TEMP_NSG_ID} --workspace ${TEMP_LAW_RESOURCE_ID} --storage-account ${TEMP_FLOWLOG_STORAGE_RESOURCE_ID}  --traffic-analytics true --interval 10
done # TEMP_NSG_ID
 
done # TEMP_SUBSCRIPTION_ID
done # VDC
 
# フローログは同一リージョンの LAW に対してしかデータ送信できない

#####################################
# Windows VM への TLS 1.2 の強制

# ■ Windows Server の通信プロトコルを TLS 1.2 に限定する
 
TEMP_TARGET_SUBSCRIPTION_IDS=$SUBSCRIPTION_IDS
for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
 
# サブスクリプションによって作業アカウントを切り替え
if [[ $TEMP_SUBSCRIPTION_ID == $SUBSCRIPTION_ID_MGMT || $TEMP_SUBSCRIPTION_ID == $SUBSCRIPTION_ID_HUB ]]; then if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_plat_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_plat_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi fi
if [[ $TEMP_SUBSCRIPTION_ID == $SUBSCRIPTION_ID_SPOKE_A ]]; then if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokea_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokea_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi fi
if [[ $TEMP_SUBSCRIPTION_ID == $SUBSCRIPTION_ID_SPOKE_B ]]; then if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokeb_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokeb_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi fi
 
az account set -s "${TEMP_SUBSCRIPTION_ID}"
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
for TEMP_RG_NAME in $(az group list --query "[?location == '${TEMP_LOCATION_NAME}' ].name" -o tsv); do
for TEMP_VM_NAME in $(az vm list --resource-group ${TEMP_RG_NAME} --query "[?storageProfile.osDisk.osType=='Windows'].name" -o tsv); do
 
TEMP_RESOURCE_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourcegroups/${TEMP_RG_NAME}/providers/microsoft.compute/virtualmachines/${TEMP_VM_NAME}"
echo $TEMP_RESOURCE_ID
 
az rest --method post --url "https://management.azure.com${TEMP_RESOURCE_ID}/runCommand?api-version=2018-04-01" --body "{\"commandId\":\"RunPowerShellScript\",\"script\":[\"New-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 2.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 2.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 3.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 3.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\PCT 1.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\PCT 1.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\Multi-Protocol Unified Hello\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\Multi-Protocol Unified Hello\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.1\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.1\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.2\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.2\\Server' -Name 'Enabled' -Value '1' -PropertyType 'DWord' -Force | Out-Null\\r\\necho done.\"]}"
 
done #TEMP_VM_NAME
done #TEMP_RG_NAME
done #TEMP_LOCATION
done #TEMP_SUBSCRIPTION_ID
 
# Windows web servers should be configured to use secure communication protocols
# https://github.com/Azure/azure-policy/blob/master/samples/GuestConfiguration/package-samples/resource-modules/SecureProtocolWebServer/DSCResources/SecureWebServer/SecureWebServer.psm1
# エラーメッセージは以下
# There is a lower TLS protocol enabled than the minimum TLS protocol version expected on this server. The current lowest TLS version enabled on this machine is '1.1'. The expected minimum TLS protocol version is '1.2'. Displaying current status of protocols: SSL 2.0 - Absent SSL 3.0 - Absent TLS 1.0 - Absent PCT 1.0 - Absent Multi-Protocol Unified Hello - Absent TLS 1.1 - Present TLS 1.2 - Present 

#####################################
# セキュリティパッチの適用指示

# 運用管理の日常作業担当者のアカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_mgmt_ops"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_mgmt_ops@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# パッチのアセスメント
#for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_IDS; do
#az account set -s $TEMP_SUBSCRIPTION_ID
#for TEMP_VM_ID in $(az vm list --query [].id -o tsv); do
#echo ${TEMP_VM_ID}
#az rest --method post --url "${TEMP_VM_ID}/assessPatches?api-version=2021-03-01"
#done
#done
 
# パッチの適用
for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_IDS; do
az account set -s $TEMP_SUBSCRIPTION_ID
for TEMP_VM_ID in $(az vm list --query [].id -o tsv); do
echo ${TEMP_VM_ID}
az rest --method post --url "${TEMP_VM_ID}/installPatches?api-version=2022-03-01" --body "{ \"maximumDuration\": \"PT4H\", \"rebootSetting\": \"IfRequired\", \"windowsParameters\": { \"classificationsToInclude\": [ \"Critical\", \"Security\", \"UpdateRollUp\", \"FeaturePack\", \"ServicePack\", \"Definition\", \"Tools\", \"Updates\" ] }, \"linuxParameters\": { \"classificationsToInclude\": [ \"Critical\", \"Security\", \"Other\" ] } }"
done
done

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
TEMP_SUBSCRIPTION_IDS=$SUBSCRIPTION_IDS

# この後でリソース診断ログ作成スクリプト（01_04_★リソース診断ログ出力の一括設定）を流してください。

```
