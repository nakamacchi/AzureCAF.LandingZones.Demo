# AppInsights 作成と有効化

Azure の監視は、ざっくり「アプリ」と「インフラ」に分けて行います。（厳密なことを言うと、両者にはオーバラップする部分がありますが、まずはざっくり以下のように理解するとよいでしょう）

- アプリの監視は Application Insights を利用する
- インフラの監視は Azure Monitor を利用する

![picture 1](./images/843d40115e791e5407316e8c05bc2eea193448a5f092f95f781bbd12a3ba95ec.png)  

ここでは、先のステップで配置した .NET Web アプリを監視できるようにするため、Application Insights のセットアップを行います。行う作業は以下の 2 つです。

- Application Insights のリソースの作成
- 自動インストルメンテーション機能の有効化

「自動インストルメンテーション機能」とは、すでに作成済み・配置済みのアプリケーションに対して各種の計測機能を織り込んでいくというもので、これを行うことにより、.NET アプリケーションの内部の深くまで踏み込んだデータ解析が可能になります。こうした解析機能は、以前はアプリケーションのコンパイル時に組み込むことが多かったのですが、現在はすでにコンパイル済みのバイナリファイルに対してでも、自動インストルメンテーション機能を利用することで、概ね同等の機能を利用できるようになっています。

```bash

# 業務システム A チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokea_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokea_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

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

done # TEMP_LOCATION

```
