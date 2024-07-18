
```bash

# ==============================
# CAE の作成

# ワークロードプロファイル環境として作成する（UDR によるロックダウンはワークロードプロファイル環境でのみ可能）
# ロックダウンでは MCR への経路開放が必要
# https://learn.microsoft.com/ja-jp/cli/azure/containerapp/env?view=azure-cli-latest#az-containerapp-env-create
# https://learn.microsoft.com/ja-jp/cli/azure/containerapp/env/workload-profile?view=azure-cli-latest

# 権限リークしている。Network 権限がなくても CAE を VNET join させられてしまう。

# 業務システム D チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spoked_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
 
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_CAE_NAME="cae-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-spoked-${TEMP_LOCATION_PREFIX}"

TEMP_CAE_SUBNET_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/virtualNetworks/${TEMP_VNET_NAME}/subnets/ContainerAppsSubnet"

# Subnet を ContainerAppsEnvironment に委任できるように設定 (Workload Profile に必要)
az network vnet subnet update --ids "${TEMP_CAE_SUBNET_ID}" --delegations "Microsoft.App/environments"

az containerapp env create --resource-group "${TEMP_RG_NAME}" --name "${TEMP_CAE_NAME}" --location ${TEMP_LOCATION_NAME} --infrastructure-subnet-resource-id "${TEMP_CAE_SUBNET_ID}" --internal-only true --logs-destination azure-monitor --enable-workload-profiles

# 占有型マシンを追加する場合
az containerapp env workload-profile add --resource-group "${TEMP_RG_NAME}" --name "${TEMP_CAE_NAME}" --workload-profile-name "wp-d4" --max-nodes 1 --min-nodes 1 --workload-profile-type "D4"

# CAE のプロビジョニングの完了を待機
while [ $(az containerapp env show --resource-group "${TEMP_RG_NAME}" --name "${TEMP_CAE_NAME}" --query properties.provisioningState -o tsv) != "Succeeded" ]
do
echo "CAE provisioning State is $(az containerapp env show --resource-group "${TEMP_RG_NAME}" --name "${TEMP_CAE_NAME}" --query properties.provisioningState -o tsv) ..."
sleep 10
done

# ==============================
# CA の作成

TEMP_CA_NAME="ca-spoked-${TEMP_LOCATION_PREFIX}"

# --ingress external にすると、VNET 内に対して公開を行う
# （--ingress internal にすると、ingress サービスが構成されず、k8s 内からしかアクセスできなくなる）
az containerapp create \
  --resource-group "${TEMP_RG_NAME}" \
  --name "${TEMP_CA_NAME}" \
  --target-port 80 \
  --ingress external \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --environment "${TEMP_CAE_NAME}" \
  --workload-profile-name "wp-d4"

#   --workload-profile-name "Consumption"

done # TEMP_LOCATION

```

