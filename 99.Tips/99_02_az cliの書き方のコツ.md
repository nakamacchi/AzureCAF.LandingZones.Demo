# az cli の書き方のコツ

## ID 値がスペースを含んでいる場合

いったん JSON で取り出し、スペースをダミー文字列に変換してから再処理する。

```bash
# スペースを加味して処理するために...
# for TEMP_RESOURCE_ID in $(az resource list --query [].id -o tsv); do
TEMP_IDS=$(az resource list --query [].id)
TEMP_IDS=${TEMP_IDS//  \"/\"}
TEMP_IDS=${TEMP_IDS//[\[\],]/}
TEMP_IDS=${TEMP_IDS// /SPACEFIX}
for TEMP in $TEMP_IDS; do
  TEMP_ID="${TEMP//SPACEFIX/ }"
  TEMP_RESOURCE_ID="${TEMP_ID//\"/}"
 
 
for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_IDS; do
  az account set -s $TEMP_SUBSCRIPTION_ID
  TEMP_IDS=$(az monitor action-group list --query [].id)
  TEMP_IDS=${TEMP_IDS//  \"/\"}
  TEMP_IDS=${TEMP_IDS//[\[\],]/}
  TEMP_IDS=${TEMP_IDS// /SPACEFIX}
  # echo $TEMP_IDS
  for TEMP in $TEMP_IDS; do
    TEMP_ID="${TEMP//SPACEFIX/ }"
    TEMP_ID="${TEMP_ID//\"/}"
    echo $TEMP_ID
    az rest --method delete --uri "${TEMP_ID}?api-version=2023-01-01"
    # az monitor action-group delete --ids "$TEMP_ID"
  done
done
```

## 変数を使った検索

- 外側は "、変数は ' で囲む。
- [? ] の内側に適宜スペースを挟む。

```bash
az vm list --query "[? location == '${TEMP_LOCATION_NAME}' ].id" --subscription ${SUBSCRIPTION_ID_SPOKE_A}
```
