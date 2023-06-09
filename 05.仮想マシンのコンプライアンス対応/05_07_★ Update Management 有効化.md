# ★ Update Management 有効化

```bash
 
# 各 VM の構成設定を変更し、メンテナンス構成（メンテナンス時間など）を設定
 
for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
echo "Setting Update Management... ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
# 当該リージョンのリソースグループを拾って処理
for TEMP_RG_NAME in $(az group list --query "[?location == '${TEMP_LOCATION_NAME}' ].name" -o tsv); do
 
# Windows マシンへ適用
for TEMP_VM_NAME in $(az vm list --resource-group ${TEMP_RG_NAME} --query "[?storageProfile.osDisk.osType=='Windows'].name" -o tsv); do
  echo ${TEMP_VM_NAME}
  az rest --method patch --url "https://management.azure.com/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}?api-version=2021-03-01" --body "{ \"location\": \"${TEMP_LOCATION_NAME}\", \"properties\": { \"osProfile\": { \"windowsConfiguration\": { \"patchSettings\": { \"assessmentMode\": \"AutomaticByPlatform\", \"patchMode\": \"AutomaticByPlatform\" } } } } }"
done
 
# Linux マシンへ適用
for TEMP_VM_NAME in $(az vm list --resource-group ${TEMP_RG_NAME} --query "[?storageProfile.osDisk.osType=='Linux'].name" -o tsv); do
  echo ${TEMP_VM_NAME}
  az rest --method patch --url "https://management.azure.com/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}?api-version=2021-03-01" --body "{ \"location\": \"${TEMP_LOCATION_NAME}\", \"properties\": { \"osProfile\": { \"linuxConfiguration\": { \"patchSettings\": { \"assessmentMode\": \"AutomaticByPlatform\", \"patchMode\": \"AutomaticByPlatform\" } } } } }"
done
 
# メンテナンス構成の作成
 
TEMP_MC_NAME="mc-daily-patching"
 
# 翌日の夜 1 時から有効になるメンテナンス構成を作成
TEMP_DATE=$(date "+%Y-%m-%d" -d "1 days")
 
cat <<EOF > tmp.json
{
  "\$schema": " https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Maintenance/maintenanceConfigurations",
      "apiVersion": "2021-09-01-preview",
      "name": "${TEMP_MC_NAME}",
      "location": "${TEMP_LOCATION_NAME}",
      "properties": {
        "maintenanceScope": "InGuestPatch",
        "installPatches": {
          "linuxParameters": {
            "classificationsToInclude": [
              "Critical",
              "Security"
            ],
            "packageNameMasksToExclude": null,
            "packageNameMasksToInclude": null
          },
          "windowsParameters": {
            "classificationsToInclude": [
              "Critical",
              "Security"
            ],
            "kbNumbersToExclude": null,
            "kbNumbersToInclude": null
          },
          "rebootSetting": "RebootIfRequired"
        },
        "extensionProperties": {
          "InGuestPatchMode": "User"
        },
        "maintenanceWindow": {
          "startDateTime": "${TEMP_DATE} 01:00",
          "duration": "03:55",
          "timeZone": "Tokyo Standard Time",
          "expirationDateTime": null,
          "recurEvery": "1Day"
        }
      }
    }
  ]
}
EOF
az deployment group create --name ${TEMP_MC_NAME} --resource-group ${TEMP_RG_NAME} --template-file tmp.json
 
# VM をメンテナンス構成へ関連付ける
 
for TEMP_VM_NAME in $(az vm list --resource-group ${TEMP_RG_NAME} --query [].name -o tsv); do
  echo ${TEMP_VM_NAME}
  az rest --method put --url "https://management.azure.com/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.Maintenance/configurationAssignments/${TEMP_MC_NAME}Assignment?api-version=2021-09-01-preview" --body "{ \"properties\": { \"maintenanceConfigurationId\": \"/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourcegroups/${TEMP_RG_NAME}/providers/Microsoft.Maintenance/maintenanceConfigurations/${TEMP_MC_NAME}\" }, \"location\": \"${TEMP_LOCATION_NAME}\" }"
done
 
done # TEMP_RG_NAME
done # TEMP_LOCATION_NAME
done # TEMP_SUBSCRIPTION_ID

```
