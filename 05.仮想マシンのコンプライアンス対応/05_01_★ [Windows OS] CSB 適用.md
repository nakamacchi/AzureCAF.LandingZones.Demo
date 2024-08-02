# ★ [Windows OS] CSB 適用

GC CSB を適用し、Windows OS をハードニングします。また、GC に関連するルールもセットアップします。

- GC に付随して適用するルールは以下の 4 つです。
  - AuditSecureProtocol (TLS 1.2 以外のプロトコルが利用不可になっているかを確認)
  - AzureWindowsBaseline (CSB によるハードニングを確認)
  - AzureWindowsVMEncryptionCompliance (BitLocker 暗号化(ADE) を確認)
  - WindowsDefenderExploitGuard (Windows Defender Exploit Guard が有効かを確認)
- CSB の適用に関しては以下の注意が必要です。
  - 既定で GC エージェントをセットアップした場合、CSB は MonitorOnly モードでインストールされます（＝チェックはするが是正はしない）。自動是正をするためには、ApplyAndMonitor モードでインストールする必要があります。
  - すでに CSB ルールがインストールされている場合、設定のみを再変更することができないため、一度既存のルールを削除し、ApplyAndMonitor モードで再インストールする必要があります。これを行っているのが下記のスクリプトです。
  - CSB ルール割り当てはサーバ側で行われるため、VM 側では即時にこれを検知できません（定期的なポーリングにより割り当てを再確認します）。急ぎで適用したい場合には、VM を再起動してください。再起動により GC エージェントがルール割り当てを確認してチェックを行います。
  - CSB ルールによるハードニングは一発でうまくいかない場合があります。その場合には本スクリプトをもう一度実行してみてください。
  - CSB によるハードニング処理とその結果報告には、少なくとも 10～30 分程度かかります。適用結果は Azure ポータルの "Guest Assginment" （ゲスト割り当て）のページから確認することができます。

```bash

for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
echo "Setting GC CSB... ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
for TEMP_RG_NAME in $(az group list --query "[?location == '${TEMP_LOCATION_NAME}' ].name" -o tsv); do
 
# Azure Windows Baseline の強制適用
for TEMP_VM_NAME in $(az vm list --resource-group ${TEMP_RG_NAME} --query "[?storageProfile.osDisk.osType=='Windows'].name" -o tsv); do
 
if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='AzurePolicyforWindows']" -o tsv)" ]; then
  echo "Installing AzurePolicyforWindows to ${TEMP_VM_NAME}..."
  az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "ConfigurationforWindows" --publisher "Microsoft.GuestConfiguration" --extension-instance-name "AzurePolicyforWindows" --enable-auto-upgrade false --no-auto-upgrade-minor-version true
fi
 
TEMP_GC_ASSIGNMENT_NAME="AzureWindowsBaseline"
TEMP_URI="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.GuestConfiguration/guestConfigurationAssignments/${TEMP_GC_ASSIGNMENT_NAME}?api-version=2022-01-25"

# 既存のものがある場合、一回消さないと割り当てられない
echo "Checking existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
TEMP=$(az rest --uri ${TEMP_URI} --method GET 2>&1)
if [[ ${TEMP} =~ "Not Found" ]]; then
echo "Not existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
else
echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --uri ${TEMP_URI} --method DELETE
fi

echo "Creating GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --method PUT --uri $TEMP_URI --body @- <<EOF
{
  "properties": {
    "guestConfiguration": {
      "name": "${TEMP_GC_ASSIGNMENT_NAME}",
      "version": "1.*",
      "assignmentType": "ApplyAndMonitor",
      "configurationParameter": [
      ],
      "configurationSetting": {
        "actionAfterReboot": "",
        "allowModuleOverwrite": false,
        "configurationMode": "ApplyAndMonitor",
        "configurationModeFrequencyMins": 15,
        "rebootIfNeeded": true,
        "refreshFrequencyMins": 5
      }
    }
  }
}
EOF
 
# 割り当たったことを確認したい場合は
# az rest --uri ${TEMP_URI} --method GET

# 他の 3 つのルールを割り当てる

# AuditSecureProtocol
TEMP_GC_ASSIGNMENT_NAME="AuditSecureProtocol"
TEMP_URI="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.GuestConfiguration/guestConfigurationAssignments/${TEMP_GC_ASSIGNMENT_NAME}?api-version=2022-01-25"

# 既存のものがある場合には消してから割り当て
echo "Checking existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
TEMP=$(az rest --uri ${TEMP_URI} --method GET 2>&1)
if [[ ${TEMP} =~ "Not Found" ]]; then
echo "Not existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
else
echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --uri ${TEMP_URI} --method DELETE
fi

echo "Creating GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --method PUT --uri $TEMP_URI --body @- <<EOF
{
  "properties": {
    "guestConfiguration": {
      "name": "${TEMP_GC_ASSIGNMENT_NAME}",
      "version": "1.*",
      "assignmentType": null,
      "configurationParameter": [
        {
          "name": "[SecureWebServer]s1;MinimumTLSVersion",
          "value": "1.2"
        }
      ],
      "configurationSetting": {
        "actionAfterReboot": "",
        "allowModuleOverwrite": false,
        "configurationMode": "MonitorOnly",
        "configurationModeFrequencyMins": 15,
        "rebootIfNeeded": true,
        "refreshFrequencyMins": 5
      }
    }
  }
}
EOF
 
# AzureWindowsVMEncryptionCompliance
TEMP_GC_ASSIGNMENT_NAME="AzureWindowsVMEncryptionCompliance"
TEMP_URI="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.GuestConfiguration/guestConfigurationAssignments/${TEMP_GC_ASSIGNMENT_NAME}?api-version=2022-01-25"

# 既存のものがある場合には消してから割り当て
echo "Checking existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
TEMP=$(az rest --uri ${TEMP_URI} --method GET 2>&1)
if [[ ${TEMP} =~ "Not Found" ]]; then
echo "Not existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
else
echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --uri ${TEMP_URI} --method DELETE
fi

echo "Creating GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --method PUT --uri $TEMP_URI --body @- <<EOF
{
  "properties": {
    "guestConfiguration": {
      "name": "${TEMP_GC_ASSIGNMENT_NAME}",
      "version": "1.*",
      "assignmentType": null,
      "configurationParameter": [],
      "configurationSetting": {
        "actionAfterReboot": "",
        "allowModuleOverwrite": false,
        "configurationMode": "MonitorOnly",
        "configurationModeFrequencyMins": 15,
        "rebootIfNeeded": true,
        "refreshFrequencyMins": 5
      }
    }
  }
}
EOF
 
# WindowsDefenderExploitGuard
TEMP_GC_ASSIGNMENT_NAME="WindowsDefenderExploitGuard"
TEMP_URI="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.GuestConfiguration/guestConfigurationAssignments/${TEMP_GC_ASSIGNMENT_NAME}?api-version=2022-01-25"

# 既存のものがある場合には消してから割り当て
echo "Checking existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
TEMP=$(az rest --uri ${TEMP_URI} --method GET 2>&1)
if [[ ${TEMP} =~ "Not Found" ]]; then
echo "Not existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
else
echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --uri ${TEMP_URI} --method DELETE
fi

echo "Creating GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --method PUT --uri $TEMP_URI --body @- <<EOF
{
  "properties": {
    "guestConfiguration": {
      "name": "${TEMP_GC_ASSIGNMENT_NAME}",
      "version": "1.*",
      "assignmentType": null,
      "configurationParameter": [
        {
          "name": "[WindowsDefenderExploitGuard]WindowsDefenderExploitGuard1;NotAvailableMachineState",
          "value": "Compliant"
        }
      ],
      "configurationSetting": {
        "actionAfterReboot": "",
        "allowModuleOverwrite": false,
        "configurationMode": "MonitorOnly",
        "configurationModeFrequencyMins": 15,
        "rebootIfNeeded": true,
        "refreshFrequencyMins": 5
      }
    }
  }
}
EOF

# VM を再起動
echo "Restarting ${TEMP_VM_NAME}"
az vm restart --force --resource-group ${TEMP_RG_NAME} --name ${TEMP_VM_NAME} --no-wait
 
done # TEMP_VM_NAME
done # TEMP_RG_NAME
done # TEMP_LOCATION_NAME
done # TEMP_SUBSCRIPTION_ID

```
