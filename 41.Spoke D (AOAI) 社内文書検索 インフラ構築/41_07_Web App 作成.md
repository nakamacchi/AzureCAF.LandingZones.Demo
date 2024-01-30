# Web App 作成

アプリケーションを配置するための Web App を作成します。

- 今回利用するアプリケーションは、以下の 2 つから構成されるアプリケーションです。
  - フロントエンド : JavaScript で作られた SPA 型 Web アプリ
  - バックエンド : Python Flask で作られた Web API アプリ
  - 実際のデプロイ時は、フロントエンドアプリをバックエンドアプリの中にコピーして、バックエンドアプリを Web App 上に配置します。
- 上記のアプリを、Azure の CaaS/PaaS サービスである Web App 上に配置して動作させます。
- 本アプリケーションは、Windows 版 Web App では稼働しません。
  - Windows 版 Web App は Python 3.4 までしかサポートしていないためです。（今回のアプリは 3.10 以上が必要）
  - このため、Linux 版 Web App か、あるいは App Service for Container のいずれかを利用する必要があります。
- ワーカーロールのサイズはやや大きめをおすすめします。
  - 今回は P2V2（2コア 7GB）マシンにしています。（Python アプリの起動を早くするため）

```bash

# 業務システム D チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spoked_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
az account set -s "${SUBSCRIPTION_ID_SPOKE_D}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_ASP_LINUX_NAME="aspl-spoked-${TEMP_LOCATION_PREFIX}"
TEMP_RG_NAME="rg-spoked-${TEMP_LOCATION_PREFIX}"

# App Service Plan (Linux) の作成
# ※ Windows 版は Python が 3.4 までしかサポートされていないため、今回のケースでは利用できない
# ※ Python 3.10 を利用するために Linux 版の App Service Plan を利用する。
TEMP_ASP_OPTIONS=$( [[ "$FLAG_USE_WORKLOAD_AZ" = true ]] && echo "--sku P2V2 --number-of-workers 3 --zone-redundant" || echo "--sku P2V2 --number-of-workers 1" )
az appservice plan create --name "${TEMP_ASP_LINUX_NAME}" --resource-group "$TEMP_RG_NAME" --location "${TEMP_LOCATION_NAME}" --is-linux $TEMP_ASP_OPTIONS

# Web App の作成
# Web App 名はグローバルに一意である必要があるため UNIQUE_SUFFIX を付与する
TEMP_WEBAPP_NAME="webapp-spoked-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_ASP_LINUX_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Web/serverFarms/${TEMP_ASP_LINUX_NAME}"
az webapp create --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME --plan $TEMP_ASP_LINUX_ID --runtime "python|3.10"

# システム割当 MID を有効化しておく
az webapp identity assign --name $TEMP_WEBAPP_NAME --resource-group $TEMP_RG_NAME

# ログの有効化 (App Service Log)
az webapp log config --name ${TEMP_WEBAPP_NAME} --resource-group ${TEMP_RG_NAME} --application-logging filesystem --detailed-error-messages true --failed-request-tracing true --web-server-logging filesystem --level warning

done # TEMP_LOCATION

```
