# DevBox プールの VM 管理

DevBox プールの中に作成された VM を確認するには、以下の方法があります。2023/09 時点では、ARM API を通して簡単に VM の一覧を取得する方法が提供されていないため、下記の方法を組み合わせて利用する必要があります。

- プール内の VM 数を確認する
- プール内の VM の一覧を確認する
- プール内の VM の利用状況を確認する
- プール内での VM の作成と削除の履歴を確認する
- 特定のユーザが作成した VM を管理者が強制削除する

## プール内の VM 数を確認する

プール内の VM 総数は、Azure Portal またはコマンドラインから確認できます。（プールごとに VM 総数は確認できますが、残念ながら VM の一覧取得などはできません。）

```bash

if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_projectx_admin@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi

TEMP_LOCATION_NAME=${LOCATION_NAMES[0]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}
TEMP_RG_NAME="rg-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_DC_NAME="dc-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_PRJ_NAME="DevProjectX"

az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.DevCenter/projects/${TEMP_PRJ_NAME}/pools?$top=50&$skip=0&api-version=2023-06-01-preview"

```

![picture 0](./images/ccec1dcc924120248a7f7960dc615704ed073c75969972e95ebfc7a20793f190.png)  

## プール内の VM の一覧を確認する

Azure Portal からはプール内の VM 一覧を取得する機能がありませんが、デバイス管理ツールである Intune から確認することができます。これは DevBox VM は Windows 365 Cloud PC としてデバイス管理ツールである Intune に登録されているためです。

![picture 1](./images/4699a570936b2a65765ebae2bea8673cb759b328b787e3b237a1dcefc038f37b.png)  

## プール内の VM の利用状況を確認する

また Intune の機能として、誰がどのデバイスを利用したのかの情報も表示することができます。（デバイス > クラウド PC パフォーマンス > クラウド PC 使用率 から確認できます）

![picture 2](./images/c6a17f824e18bd9c29b88bdb21c92f7d3046cc02d361470a6eddfaf69be84e7a.png)  

## プール内での VM の作成と削除の履歴を確認する

前述した Intune の機能は、セキュリティ管理者でないとアクセスすることができません。このため、開発プロジェクトの管理者がプール内の VM の一覧を取得したい場合には、（現状では）DevCenter から出力される診断ログを検索するのがよいでしょう。

```KQL クエリ

DevCenterDiagnosticLogs
| where TimeGenerated > ago(30d)
| extend Parts = split(TargetResourceId, "/")
| extend DevProject = Parts[6], UserName = Parts[8], Type = Parts[9],
    VMName = split(Parts[10], ":")[0], OpName2 = split(Parts[10], ":")[1]
| extend Parts2 = split(OperationName, "/")
| extend OpName = Parts2[4]
| extend ParsedCallerIdentity = parse_json(CallerIdentity)
| extend Identity = tostring(ParsedCallerIdentity.ObjectID[0].Identity)
| where Type =~ "DEVBOXES"
| project TimeGenerated, DevProject, UserName, Type, VMName, OpName, OpName2, Identity, OperationResult

```

| TimeGenerated [UTC]      | DevProject  | UserName | Type     | VMName | OpName | OpName2 | Identity                              | OperationResult |
|--------------------------|-------------|----------|----------|--------|--------|---------|---------------------------------------|-----------------|
| 9/13/2023, 2:39:57.625 PM| DEVPROJECTY | ME       | DEVBOXES | VM001  | delete |         | 3892760b-f673-4602-a3dd-b1f538e7b398  | Success         |
| 9/13/2023, 2:39:45.006 PM| DEVPROJECTX | ME       | DEVBOXES | VM001  | action | STOP    | 9e78363b-ea6f-4b97-a1cf-8484617b3a8f  | Success         |
| 9/14/2023, 12:02:30.864 AM|DEVPROJECTX | ME       | DEVBOXES | VM002  | write  |         | 25d9f9ad-25c6-418a-9071-2a0eaf86cc4d  | Success         |
| 9/13/2023, 1:24:04.444 PM| DEVPROJECTX | ME       | DEVBOXES | VM001  | write  |         | 9e78363b-ea6f-4b97-a1cf-8484617b3a8f  | Success         |
| 9/13/2023, 2:05:42.869 PM| DEVPROJECTY | ME       | DEVBOXES | VM001  | write  |         | 3892760b-f673-4602-a3dd-b1f538e7b398  | Success         |

## 特定のユーザが作成した VM を管理者が強制削除する

すべてのユーザが作成した VM の一覧取得は厄介ですが、特定ユーザが作成した VM の一覧の取得は DevCenter に API があるため、容易です。さらにこれらを強制削除することも可能です。それぞれ以下の API を利用します。

- [特定ユーザが作成した VM の一覧を取得する](https://learn.microsoft.com/en-us/rest/api/devcenter/developer/dev-boxes/list-dev-boxes-by-user?tabs=HTTP)
- [プール内の VM を管理者が強制削除する](https://learn.microsoft.com/en-us/rest/api/devcenter/developer/dev-boxes/delete-dev-box?tabs=HTTP)

削除に際しては、以下の情報が必要です。

- DevCenter URI
- 開発プロジェクト名
- DevBox VM の要求者の Object ID
- DevBox VM の名前

```bash

if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_projectx_admin@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi

TEMP_LOCATION_NAME=${LOCATION_NAMES[0]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}
TEMP_RG_NAME="rg-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_DC_NAME="dc-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_PRJ_NAME="DevProjectX"
TEMP_DC_URI=$(az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.DevCenter/projects/${TEMP_PRJ_NAME}?api-version=2023-04-01" --query "properties.devCenterUri" -o tsv)

TEMP_USER_NAME="user_projectx_user2@${PRIMARY_DOMAIN_NAME}"

# 当該ユーザの object id を取得
TEMP_USER_OBJ_ID=$(az ad user show --id $TEMP_USER_NAME --query id -o tsv)

# 当該ユーザが保有する VM 一覧を取得
az rest --method GET --uri "${TEMP_DC_URI}projects/${TEMP_PRJ_NAME}/users/${TEMP_USER_OBJ_ID}/devboxes?api-version=2023-04-01" --resource "https://devcenter.azure.com/"

# 全 VM を削除する場合
TEMP_VM_NAMES=$(az rest --method GET --uri "${TEMP_DC_URI}projects/${TEMP_PRJ_NAME}/users/${TEMP_USER_OBJ_ID}/devboxes?api-version=2023-04-01" --resource "https://devcenter.azure.com/" --query "value[].name" -o tsv)

for TEMP_VM_NAME in $TEMP_VM_NAMES; do

echo "Deleting ${TEMP_USER_NAME} ${TEMP_VM_NAME}..."
az rest --method DELETE --uri "${TEMP_DC_URI}projects/${TEMP_PRJ_NAME}/users/${TEMP_USER_OBJ_ID}/devboxes/${TEMP_VM_NAME}?api-version=2023-04-01" --resource "https://devcenter.azure.com/"

done # TEMP_VM_NAME

```
