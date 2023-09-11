# DevCenter の作成

DevBox, Deployment Environment を利用する前提条件となる DevCenter を作成します。

- DevCenter を有効化することにより、システム開発プロジェクトの作成・管理と、セルフサービスポータルの利用が可能になります。
- DevCenter の構造は下図の通りです。
  - DevCenter 内に、環境配置機能や開発ボックスの定義があり、これを各開発プロジェクトに取り出して利用します。
  - （現状では、開発者向けにセルフサービス提供できるものは、環境配置機能と開発ボックスの 2 種類のみです。）

![picture 0](./images/a05ba5bb9cfa61e1c0c0569b445344b11a00372a13c778ab6ad19804be28de50.png)  

ここでは DevCenter を、メインリージョンに作成します。DevCenter から各種のリソース作成が行われるため、DevCenter の Managed ID は必ず有効化してください。

```bash

# 作業アカウント・作業サブスクリプション切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_dev1_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
az account set -s "${SUBSCRIPTION_ID_DEV1}"

# メインリージョンに開発環境を作成する
TEMP_LOCATION_NAME=${LOCATION_NAMES[0]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}
 
# Dev1 DevCenter の作成
TEMP_RG_NAME="rg-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_DC_NAME="dc-devcenter-${TEMP_LOCATION_PREFIX}"

az group create --name ${TEMP_RG_NAME} --location ${TEMP_LOCATION_NAME}

# az devcenter admin devcenter create --name $TEMP_DC_NAME --resource-group $TEMP_RG_NAME --identity-type SystemAssigned --location ${TEMP_LOCATION_NAME}

az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.DevCenter/devcenters/${TEMP_DC_NAME}?api-version=2023-04-01" --body @- <<EOF
{
  "location": "${TEMP_LOCATION_NAME}",
  "identity": {
    "type": "SystemAssigned"
  },
  "properties": {}
}
EOF

```
