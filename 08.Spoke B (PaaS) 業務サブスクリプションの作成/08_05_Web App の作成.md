# Web App の作成

SQL Database に続いて、Web App (AppService) を作成します。

- Web App を利用するには、まず App Service Plan を作成します。
  - App Service Plan は、自分のアプリを載せるための「Web サーバ」に相当するものです。Web App はフロントエンドロードバランサとバックエンド Web サーバから構成されており、どこまで自分専用のものを利用するのかにおいて、3 通りの選択肢があります。
    - 独立占有型（ASE, App Service Environment と呼ばれます）を利用すると、ロードバランサからバックエンドまで自分専用のものを用意することができますが、その分非常に高くなります。開発・PoC 目的であれば共有型、運用用途であっても占有型で十分なことがほとんどです。
    - ![picture 1](./images/dd5e05136dad906ef7cc852aa5c8c1e09ab5fd143a29d1f76207f32f5355d4e9.png)  
  - App Service Plan を選択する際、サーバサイズの選択に加えて OS 種別（Windows, Linux）の選択が必要になります。載せるアプリに応じて、適切な OS 種別を選択してください。
    - なお、一つの App Service Plan に複数のユーザアプリを載せることができます。例えば P2 サイズのマシンを 3 台確保し、そこに 10 種類のユーザアプリを載せる、といった使い方ができます（すべてのアプリが 3 台に分散配置されます）。
    - 各アプリはコンテナ技術により隔離されますので、お互いのデータを読み取ったりすることはできません。（このため共有型の App Service Plan を利用しても、同一マシンに共存する他のユーザのアプリのデータを覗き見したりすることはできません。）
    - ![picture 3](./images/adbe44fc834f2e097506c84869f3a48e13575fde81eea1fc4f4d4e7a37c68a0f.png)  
- App Service Plan を作成したのち、App Service (Web App)を作成します。
  - Web App 単位に、利用するランタイムなどを選択し、アプリを配置して利用することになります。
  - Web App に対して、タイムゾーンを設定しておきます。（無指定の場合は UTC タイムゾーンの扱いになります。）

```bash

# 業務システム B チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokeb_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokeb_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# Spoke B サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_B}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_ASP_WIN_NAME="aspw-spokeb-${TEMP_LOCATION_PREFIX}"
TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"

# App Service Plan (Windows) の作成
TEMP_ASP_OPTIONS=$( [[ "$FLAG_USE_WORKLOAD_AZ" = true ]] && echo "--sku P1V2 --number-of-workers 3 --zone-redundant" || echo "--sku P1V2 --number-of-workers 1" )
az appservice plan create --name "${TEMP_ASP_WIN_NAME}" --resource-group "$TEMP_RG_NAME" --location "${TEMP_LOCATION_NAME}" $TEMP_ASP_OPTIONS

# Web App の作成
# Web App 名はグローバルに一意である必要があるため UNIQUE_SUFFIX を付与する
TEMP_WEBAPP_NAME="webapp-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_ASP_WIN_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_B}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Web/serverFarms/${TEMP_ASP_WIN_NAME}"
az webapp create --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --plan $TEMP_ASP_WIN_ID

# システム割当 MID を有効化しておく　※ 本来はこの ID でリソースアクセスするのが望ましい
az webapp identity assign --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME

# タイムゾーンの設定
az webapp config appsettings set --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --settings "WEBSITE_TIME_ZONE=Tokyo Standard Time"
# FTP の無効化 (Windows 版では Kudu で事足りるので塞ぐ)
az webapp config set --ftps-state Disabled --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME}
# ログの有効化 (App Service Log)
az webapp log config --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --application-logging filesystem --detailed-error-messages true --failed-request-tracing true --web-server-logging filesystem --level warning

done # TEMP_LOCATION

```
