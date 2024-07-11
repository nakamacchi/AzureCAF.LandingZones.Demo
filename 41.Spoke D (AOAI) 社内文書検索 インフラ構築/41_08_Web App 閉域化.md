# Web App 閉域化

先の作業で作成した Web App を閉域化します。

- 出力ロックダウン（egress lockdown）を行うため、以下を行います。
  - Web App の VNET 統合機能を利用するとともに、WEBSITE_VNET_ROUTE_ALL=1 を設定し、Web アプリからの通信をすべて VNET 側に流す。
  - Web App がプライベートエンドポイントを利用できるように、DNS サーバとして Hub の Azure Firewall を利用するように設定する。
- 入力ロックダウン (ingress lockdown) のためのプライベートエンドポイントの作成は、他のリソースとまとめて次のステップで行います。

```bash

# Web App の閉域化

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_ASP_WIN_NAME="aspw-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_WEBAPP_NAME="webapp-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_VNET_NAME="vnet-spoked-${TEMP_LOCATION_PREFIX}"

# 出力ロックダウン（Egress Lockdown）==================================
# Regional VNET Integration (VNET 統合 v2) の作成
# https://docs.microsoft.com/ja-jp/azure/app-service/web-sites-integrate-with-vnet#regional-vnet-integration
az webapp vnet-integration add --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --subnet "AppServiceBackendSubnet" --vnet ${TEMP_VNET_NAME}

TEMP_FW_IP=$(az network firewall ip-config list -g "rg-hub-${TEMP_LOCATION_PREFIX}" -f "fw-hub-${TEMP_LOCATION_PREFIX}" --query "[0].privateIpAddress" --output tsv --subscription ${SUBSCRIPTION_ID_HUB})

az webapp config appsettings set --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --settings "WEBSITE_VNET_ROUTE_ALL=1"
az webapp config appsettings set --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --settings "WEBSITE_DNS_SERVER=${TEMP_FW_IP}"

# 入力ロックダウン（Ingress Lockdown） ===============================
# Private Endpoint の作成は他とまとめて行う

# ワーカープロセスのリセット
# ネットワーク関連の設定を変えたらアプリケーションをリセット（アプリケーションリスタートではなくプロセスリセットをしないと反映されないため）
az webapp restart --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME}

done # TEMP_LOCATION

```
