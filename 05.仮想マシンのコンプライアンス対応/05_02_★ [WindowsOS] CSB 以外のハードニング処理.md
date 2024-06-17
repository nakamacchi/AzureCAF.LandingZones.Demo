# ★ [WindowsOS] CSB 以外の OS ハードニング処理

以下の処理は CSB のハードニング処理には含まれないため、runCommand API を利用して VM 内にスクリプトを送り込み、是正処理を行います。

- ゲストアカウントのリネーム
- Windows Defender Exploit Guard の有効化
- TLS 1.2 の強制

なお、本来、リネーム先は乱数生成などにより VM 単位に変えるのが適切ですが、ここでは固定的に "LocalGuest" という名前にしています。より堅牢にするためには変更することをオススメします。

```bash

for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
az account set -s "${TEMP_SUBSCRIPTION_ID}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# 当該リージョンのリソースグループを拾って処理
for TEMP_RG_NAME in $(az group list --query "[?location == '${TEMP_LOCATION_NAME}' ].name" -o tsv); do

for TEMP_VM_NAME in $(az vm list --resource-group ${TEMP_RG_NAME} --query "[?storageProfile.osDisk.osType=='Windows'].name" -o tsv); do
  TEMP_RESOURCE_ID=$(az vm show --resource-group ${TEMP_RG_NAME} --name ${TEMP_VM_NAME} --query id -o tsv)

  echo "Hardning Windows OS on ${TEMP_VM_NAME}..."
  az rest --method post --url "https://management.azure.com${TEMP_RESOURCE_ID}/runCommand?api-version=2018-04-01" --body "{\"commandId\":\"RunPowerShellScript\",\"script\":[\"wmic useraccount where \\\"name='Guest'\\\" rename 'LocalGuest'\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids be9ba2d9-53ea-4cdc-84e5-9b1eeee46550 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids b2b3f03d-6a65-4f7b-a9c7-1c7ef74a9ba4 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 9e6c4e1f-7d60-472f-ba1a-a39ef669e4b2 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids d4f940ab-401b-4efc-aadc-ad5f3c50688a -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids d3e037e1-3eb8-44c8-a917-57927947596d -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 5beb7efe-fd9a-4556-801d-275e5ffc04cc -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 3b576869-a4ec-4529-8536-b80a7769e899 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 26190899-1602-49e8-8b27-eb1d0a1ce869 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 92e97fa1-2edf-4476-bdd6-9dd0b4dddc7b -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 7674ba52-37eb-4a4f-a9a1-f0f9a1619a2c -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 75668c1f-73b5-4cf0-bb93-3ecf5cb7cc84 -AttackSurfaceReductionRules_Actions Enabled\\r\\nSet-MpPreference -EnableControlledFolderAccess Enabled\\r\\necho done.\\r\\nwmic useraccount where \\\"name='Guest'\\\" rename 'LocalGuest'\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 2.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 2.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 3.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 3.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\PCT 1.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\PCT 1.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\Multi-Protocol Unified Hello\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\Multi-Protocol Unified Hello\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.1\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.1\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.2\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.2\\Server' -Name 'Enabled' -Value '1' -PropertyType 'DWord' -Force | Out-Null\\r\\necho done.\"]}"

done # TEMP_VM_NAME
done # TEMP_RG_NAME

done # TEMP_LOCATION
done # TEMP_SUBSCRIPTION

```
