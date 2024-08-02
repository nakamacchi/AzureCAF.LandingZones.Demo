# DevBox プールの利用制限

開発ボックスをむやみに作成して放置されると、お金の無駄遣いになります。このため、DevBox プールには適切な利用制限をかけておくことが必要です。

- 作成できる開発ボックスの数を制限する
- 自動的にシャットダウンする（※ 今後、RDP 接続が切れた際の自動的なサスペンドも可能になる予定）

これらの制約は、ポータルサイトからでもコマンドラインからでも設定できます。

```bash

# DevCenter 構築用アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_dev1_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_dev1_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
# Dev1 サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_DEV1}"

TEMP_LOCATION_NAME=${LOCATION_NAMES[0]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}
TEMP_RG_NAME="rg-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_DC_NAME="dc-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_PRJ_NAME="DevProjectX"
TEMP_DBP_NAME="dbp-devprojectx-vs2022-8core-${TEMP_LOCATION_PREFIX}"

# 開発ボックス数制限：各開発者が当該開発プロジェクト内に作成できる VM 数を制限できる
az rest --method PATCH --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.DevCenter/projects/${TEMP_PRJ_NAME}?api-version=2023-04-01" --body @- << EOF
{
  "properties" : {
    "maxDevBoxesPerUser": 3
  }
}
EOF

# 自動シャットダウン（自動起動も指定可能）
az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.DevCenter/projects/${TEMP_PRJ_NAME}/pools/${TEMP_DBP_NAME}/schedules/default?api-version=2023-04-01" --body @- <<EOF
{
  "properties": {
    "state": "Enabled",
    "type": "StopDevBox",
    "frequency": "Daily",
    "timeZone": "Asia/Tokyo",
    "time": "21:00"
  }
}
EOF

```
