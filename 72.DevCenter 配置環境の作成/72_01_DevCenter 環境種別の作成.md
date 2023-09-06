# DevCenter 環境種別の作成

まずは DevCenter 管理者が、DevCenter 環境種別を定義しておきます。

- DevCenter 環境種別は、DevCenter 全体で環境種別名に一貫性を持たせるためのものです。
  - システム的に定義するのは環境種別の「名前」のみです。
  - 環境種別の名称を決める際は、会社全体としての「環境の使い方ルール」に基づいて決めるようにします。例えば...
    - Prod : 本番環境
    - Staging : ステージング環境
    - QA : テスト環境（※ 皆で共用して利用するテスト環境。IT, ST などで利用する。）
    - Lab : ラボ環境（※ 個々人に払い出される小型のテスト環境）
    - Dev : 開発環境（開発中のデバッグなどに利用する環境）
    - Sandbox : お試しでスクラップ＆ビルドする、開発・テストなどからは隔離された環境
  - これにより、環境名を見たときに用途がすぐにわかるようにしておきます。
- ここでは DevCenter 環境種別として Sandbox, Lab の 2 種類を定義します。
  - 現状では "Prod" "Staging" などに向けて提供されている DevCenter 機能はないため、"Sandbox" "Dev" "Lab" などを定義しておけば十分です。
  - 本デモでは、プロジェクト環境種別として "Sandbox" のみを利用します。（が、それだと寂しいので DevCenter 環境種別としては "Lab" も定義しておきます）

```bash

if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_dev1_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
az account set -s "${SUBSCRIPTION_ID_DEV1}"

TEMP_LOCATION_NAME=${LOCATION_NAMES[0]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}
TEMP_RG_NAME="rg-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_DC_NAME="dc-devcenter-${TEMP_LOCATION_PREFIX}"

for TEMP_ENVTYPE_NAME in "Sandbox" "Lab" ; do

az rest --method PUT --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.DevCenter/devcenters/${TEMP_DC_NAME}/environmentTypes/${TEMP_ENVTYPE_NAME}?api-version=2023-04-01" --body @- <<EOF
{}
EOF

done #TEMP_ENVTYPE_NAME

```
