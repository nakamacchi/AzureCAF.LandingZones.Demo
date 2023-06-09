# ★ [WindowsOS] ゲストアカウントのリネーム

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
 
# Windows OS の Guest Account 名を変更（リネーム先の名前がないため、CSB では自動補正できない）
  echo "Rename Guest Account to another name on ${TEMP_VM_NAME}..."
  az rest --method post --url "https://management.azure.com${TEMP_RESOURCE_ID}/runCommand?api-version=2018-04-01" --body "{\"commandId\":\"RunPowerShellScript\",\"script\":[\"wmic useraccount where \\\"name='Guest'\\\" rename 'LocalGuest'\"]}"
 
done # TEMP_VM_NAME
done # TEMP_RG_NAME
 
done # TEMP_LOCATION
done # TEMP_SUBSCRIPTION
 
```
