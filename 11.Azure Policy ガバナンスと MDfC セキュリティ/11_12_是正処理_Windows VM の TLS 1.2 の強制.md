# 是正処理 : Windows VM の TLS 1.2 の強制

Windows マシン用の GC のルールの一つである、AuditSecureProtocol ルールの是正方法を示します。このルールは　TLS 1.2 以外のプロトコルが利用不可になっているかを確認するもので、プロトコルを無効化するためにはレジストリを設定します。

コマンドラインから実行する方法は[こちら](/99.Tips/99_03_Windows%20OS%20%E3%81%A7%E3%81%AE%20TLS%201.2%20%E5%BC%B7%E5%88%B6.md)に示していますが、VM にログインするのは面倒なため、runCommand API を使って処理します。

```bash
 
# ■ Windows Server の通信プロトコルを TLS 1.2 に限定する
 
TEMP_TARGET_SUBSCRIPTION_IDS=$SUBSCRIPTION_IDS
for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
 
# サブスクリプションによって作業アカウントを切り替え
if [[ $TEMP_SUBSCRIPTION_ID == $SUBSCRIPTION_ID_MGMT || $TEMP_SUBSCRIPTION_ID == $SUBSCRIPTION_ID_HUB ]]; then if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_plat_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi fi
if [[ $TEMP_SUBSCRIPTION_ID == $SUBSCRIPTION_ID_SPOKE_A ]]; then if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spokea_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi fi
if [[ $TEMP_SUBSCRIPTION_ID == $SUBSCRIPTION_ID_SPOKE_B ]]; then if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spokeb_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi fi
 
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

```
