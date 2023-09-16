# DevCenter での DevBox 定義の作成

DevBox 定義を作成し、当該 DevCenter 内で利用してよい VM イメージと VM のサイズの組み合わせを定義します。DevCenter 配下にある各開発プロジェクトは、ここで定義する VM イメージ・VM サイズの組み合わせから選ぶ形になります。事前に定義された組み合わせ以外での VM 作成はできません。

![picture 0](./images/a05ba5bb9cfa61e1c0c0569b445344b11a00372a13c778ab6ad19804be28de50.png)  

## VM イメージについて

- 標準で用意されている VM イメージは、2023/09 時点の場合、以下のようになります。
  - Windows 365 標準イメージ
    - Windows 10 系列
      - Windows 10 Enterprise + OS Optimizations 20H2
      - Windows 10 Enterprise + OS Optimizations 21H2
      - Windows 10 Enterprise + OS Optimizations 22H2
      - Windows 10 Enterprise + Microsoft 365 Apps 20H2
      - Windows 10 Enterprise + Microsoft 365 Apps 21H2
      - Windows 10 Enterprise + Microsoft 365 Apps 22H2
    - Windows 11 系列
      - Windows 11 Enterprise + OS Optimizations 21H2
      - Windows 11 Enterprise + OS Optimizations 22H2
      - Windows 11 Enterprise + Microsoft 365 Apps 21H2
      - Windows 11 Enterprise + Microsoft 365 Apps 22H2
      - Windows 11 Enterprise + Developer Optimizations 22H2
  - DevBox 専用イメージ
    - Windows 10 + Visual Studio Professional 系列
      - Visual Studio 2019 Pro on Windows 10 Enterprise + Microsoft 365 Apps 22H2
      - Visual Studio 2022 Pro on Windows 10 Enterprise + Microsoft 365 Apps 22H2
    - Windows 10 + Visual Studio Enterprise 系列
      - Visual Studio 2019 Enterprise on Windows 10 Enterprise + Microsoft 365 Apps 22H2
      - Visual Studio 2022 Enterprise on Windows 10 Enterprise + Microsoft 365 Apps 22H2
    - Windows 11 + Visual Studio Professional 系列
      - Visual Studio 2019 Pro on Windows 11 Enterprise + Microsoft 365 Apps 22H2
      - Visual Studio 2022 Pro on Windows 11 Enterprise + Microsoft 365 Apps 22H2
    - Windows 11 + Visual Studio Enterprise 系列
      - Visual Studio 2019 Enterprise on Windows 11 Enterprise + Microsoft 365 Apps 22H2
      - Visual Studio 2022 Enterprise on Windows 11 Enterprise + Microsoft 365 Apps 22H2
- 上記のイメージについて、以下の点を知っておくとよいでしょう。
  - "OS Optimizations" とは、Azure 仮想環境上での動作用に最適化された VM イメージです。UWP パッケージの削除やタスクスケジューラアクションの無効化などが行われています。 
    - 詳細は[こちら](https://learn.microsoft.com/ja-jp/windows-365/enterprise/device-images#gallery-images)
    - M365 Apps（いわゆる Office）が含まれているイメージに関しても、同様の最適化が行われています。
  - Windows 365 標準イメージに加えて、Visual Studio がプレインストールされた DevBox 専用イメージが用意されています。
    - いずれも Windows 365 の M365 Apps 入りのイメージに、VS 2019/2022 の Pro/Ent がインストールされる、という形になっています。
    - 実際にインストールされているアプリ（VS, M365）を利用するためには、サインインによるライセンス認証が必要です。
- 各 OS のイメージには id と name が設定されており、コマンドラインから設定する場合にはイメージの id 値（または name 値）が必要になります。
  - 以下のコマンドにより一覧を取得することができます。
```bash
az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.DevCenter/devcenters/${TEMP_DC_NAME}/images?api-version=2023-04-01" --query "value[].[id,name,properties.description]"
```
  - 例えば全部入りイメージの場合は以下の通りです。
    - Visual Studio 2022 Enterprise on Windows 11 Enterprise + Microsoft 365 Apps 22H2
    - microsoftvisualstudio_visualstudioplustools_vs-2022-ent-general-win11-m365-gen2
  - なお、OS はいずれも英語イメージです。
    - 日本語化が必要な場合には、設定画面から変更を行います。（方法は Windows 365 の場合と同じです。）
    - Settings > Time & Language > Language & Region > Preferred languages > Add a language > 日本語 (Japanese) を追加
    - Set as my Windows display language をチェックして install

## 利用可能なマシンタイプについて

- スペック (SKU) の選択幅は 2023/09 現在、以下の通りです。
  - vCPU は 8, 16, 32 から選択（メモリは vCPU 数 × 4 GB）
  - ディスク容量は 256, 512, 1024, 2048 から選択
- SKU の指定には name 値が必要です。
  - 以下のコマンドより一覧を取得することができます。
```bash
az rest --method GET --uri "/subscriptions/${SUBSCRIPTION_ID_DEV1}/providers/Microsoft.DevCenter/skus?api-version=2023-04-01" --query value[].name
```
  - 例えば 8vCPU, 32GB, 256GB の場合は以下になります。
    - general_i_8c32gb256ssd_v2

## DevBox 定義の例について

ここでは、DevBox 定義として以下の 2 種類を作成してみます。

| DevBox 定義名 | イメージ名 | VMサイズ |
| --- | --- | --- |
| vs2022ent-8core-32gb-256gb | microsoftvisualstudio_visualstudioplustools_vs-2022-ent-general-win11-m365-gen2 | general_i_8c32gb256ssd_v2 |
| vs2022ent-16core-64gb-256gb | microsoftvisualstudio_visualstudioplustools_vs-2022-ent-general-win11-m365-gen2 | general_i_16c64gb256ssd_v2 |

```bash

# DevCenter 構築用アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_dev1_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
# Dev1 サブスクリプションで作業
az account set -s "${SUBSCRIPTION_ID_DEV1}"

TEMP_LOCATION_NAME=${LOCATION_NAMES[0]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[0]}
TEMP_RG_NAME="rg-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_DC_NAME="dc-devcenter-${TEMP_LOCATION_PREFIX}"
TEMP_DC_ID="/subscriptions/${SUBSCRIPTION_ID_DEV1}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.DevCenter/devcenters/${TEMP_DC_NAME}"

TEMP_DBD_DEFS="\
vs2022ent-8core-32gb-256gb,microsoftvisualstudio_visualstudioplustools_vs-2022-ent-general-win11-m365-gen2,general_i_8c32gb256ssd_v2 \
vs2022ent-16core-64gb-256gb,microsoftvisualstudio_visualstudioplustools_vs-2022-ent-general-win11-m365-gen2,general_i_16c64gb256ssd_v2 \
"

for TEMP_DBD_DEF in $TEMP_DBD_DEFS; do
TEMP=(${TEMP_DBD_DEF//,/ })
TEMP_DBD_NAME=${TEMP[0]}
TEMP_VM_IMAGE_NAME=${TEMP[1]}
TEMP_VM_SKU_NAME=${TEMP[2]}

az rest --method PUT --uri "${TEMP_DC_ID}/devboxdefinitions/${TEMP_DBD_NAME}?api-version=2023-04-01" --body @- <<EOF
{
  "location": "${TEMP_LOCATION_NAME}",
  "properties": {
    "imageReference": {
        "id": "${TEMP_DC_ID}/galleries/default/images/${TEMP_VM_IMAGE_NAME}"
    },
    "sku": {
        "name": "${TEMP_VM_SKU_NAME}"
    },
    "hibernateSupport": "Enabled",
  }
}
EOF

done # TEMP_DBD_DEF

```
