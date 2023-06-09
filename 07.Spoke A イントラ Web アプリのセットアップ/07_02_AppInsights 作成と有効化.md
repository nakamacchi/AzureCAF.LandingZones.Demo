# AppInsights 作成と有効化

```bash
 
# 業務システム A チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spokea_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
az account set -s "${SUBSCRIPTION_ID_SPOKE_A}"
 
# Application Insighs の作成と自動インストルメンテーション機能の有効化
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
  TEMP_RG_NAME="rg-spokea-${TEMP_LOCATION_PREFIX}"
  TEMP_APP_NAME="app-spokea-${TEMP_LOCATION_PREFIX}"
 
  TEMP_LAW_NAME="law-vdc-${TEMP_LOCATION_PREFIX}"
  TEMP_LAW_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/${TEMP_LAW_NAME}"
 
az monitor app-insights component create --app ${TEMP_APP_NAME} --location ${TEMP_LOCATION_NAME} --kind web --resource-group ${TEMP_RG_NAME} --workspace ${TEMP_LAW_RESOURCE_ID}
 
 
# https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/codeless-overview
# az cli を使ってインストール
# 接続文字列を取得し、VM extension を設定
cat <<EOF > tmp.json
{
  "redfieldConfiguration": {
    "instrumentationKeyMap": {
      "filters": [
        {
          "appFilter": ".*",
          "machineFilter": ".*",
          "virtualPathFilter": ".*",
          "instrumentationSettings" : {
            "connectionString": "$(az monitor app-insights component show --resource-group ${TEMP_RG_NAME} --app ${TEMP_APP_NAME} --query connectionString -o tsv)"
          }
        }
      ]
    }
  }
}
EOF
 
az vm extension set --resource-group ${TEMP_RG_NAME} --vm-name "vm-web-${TEMP_LOCATION_PREFIX}" --name "ApplicationMonitoringWindows" --publisher "Microsoft.Azure.Diagnostics" --version 2.8 --settings tmp.json --protected-settings '{}'
 
done

```
