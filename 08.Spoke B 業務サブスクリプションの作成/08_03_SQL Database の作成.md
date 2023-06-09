# SQL Database の作成

```bash
 
# 業務システム B チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_spokeb_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# Spoke B サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_SPOKE_B}"
 
for i in ${VDC_NUMBERS}; do
  TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
  TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}
 
# パラメータ
# （論理サーバはグローバル一意名が必要なため UNIQUE_SUFFIX を付与して名前を作る）
TEMP_SQL_SERVER_NAME="sql-spokeb-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"
TEMP_SQL_DB_NAME="pubs"
TEMP_RG_NAME="rg-spokeb-${TEMP_LOCATION_PREFIX}"
 
# SQL Server（論理サーバ）の作成
az sql server create --name $TEMP_SQL_SERVER_NAME --resource-group $TEMP_RG_NAME --location $TEMP_LOCATION_NAME --admin-user $ADMIN_USERNAME --admin-password $ADMIN_PASSWORD --enable-public-network false
 
# SQL Database の作成
az sql db create --server $TEMP_SQL_SERVER_NAME --resource-group $TEMP_RG_NAME --name $TEMP_SQL_DB_NAME --edition Basic --capacity 5
 
done

```

- SQL Server の Audit 機能を GUI から有効化
  - コマンドラインから実行する方法が現時点で不明。以下の方法ではうまく有効化できない

```bash 
# SQL Server Audit の有効化
#TEMP_MAIN_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}
#TEMP_LAW_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_MAIN_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/law-vdc-${TEMP_MAIN_LOCATION_PREFIX}"
#az sql server audit-policy update --resource-group $TEMP_RG_NAME --name $TEMP_SQL_SERVER_NAME --state Enabled --lats Enabled --lawri ${TEMP_LAW_ID}
```

![picture 1](./images/7b96001ade7b42feab63d1920f433019a1bccb91752570cc761f1296547821a1.png)  

