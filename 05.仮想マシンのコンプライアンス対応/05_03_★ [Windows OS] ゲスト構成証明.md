# ★ [Windows OS] ゲスト構成証明

ゲスト構成証明（Guest Attestation）とは、ホストシステムが仮想マシンやコンテナなどのゲストシステムの状態を検証し、その信頼性を確認するものです。ゲストシステムの起動プロセス、OS の状態などを元に、不正に改ざんされていないことを確認します。典型的には Confidential Computing（機密コンピューティング）において、機密環境が適切に構成されていることを確認する目的で利用されますが、通常の VM でも Trusted Launch が正しく機能していることなどの確認に用いることができます。詳細は[こちら](https://learn.microsoft.com/ja-jp/azure/virtual-machines/trusted-launch#vtpm)を参照してください。

```bash

for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
echo "Setting Guest Attestation... ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"
 
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
for TEMP_RG_NAME in $(az group list --query "[?location == '${TEMP_LOCATION_NAME}' ].name" -o tsv); do
 for TEMP_VM_NAME in $(az vm list --resource-group ${TEMP_RG_NAME} --query "[?storageProfile.osDisk.osType=='Windows'].name" -o tsv); do
 
if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='GuestAttestation']" -o tsv)" ]; then
  echo "Installing GuestAttestation to ${TEMP_VM_NAME}..."
  az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "GuestAttestation" --publisher "Microsoft.Azure.Security.WindowsAttestation" --extension-instance-name "GuestAttestation" --enable-auto-upgrade false --no-auto-upgrade-minor-version true
fi
 
done # TEMP_VM_NAME
done # TEMP_RG_NAME
done # TEMP_LOCATION_NAME
done # TEMP_SUBSCRIPTION_ID

```
