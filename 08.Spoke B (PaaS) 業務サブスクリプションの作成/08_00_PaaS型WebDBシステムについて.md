# PaaS 型 Web-DB システムについて

先のセクションでは IaaS VM を利用して Web-DB システムを構成しましたが、ベースになっている仮想マシンの準備などがかなり面倒でした。このため、特に新規開発などの場合には、PaaS 型サービスをご利用いただいてシステムを構築することをオススメします。特に、Azure には Web App (App Service) と SQL Database という非常に便利な PaaS サービスがあります。

- App Service （Web App） = クラスタリングされた、自動メンテナンスつき Web サーバ
- SQL Database = クラスタリングされた、自動メンテナンスつき SQL Server

どちらのサービスも Azure の黎明期から提供されており、すでに 10 年以上の稼働実績がありますが、現在でも解析機能などをはじめとしたサービス拡充が行われ続けている Azure の超重要サービスです。PaaS サービスであるため、基盤（OS・ミドルウェア）部分はマイクロソフトにお任せすることができ、ユーザ側はアプリやデータの部分に注力することができます。……といっても裏側で動作をしているのは IaaS VM だったりしますので、どんなものかを理解するには、下図のような昔ながらのサーバ構成図のようなものの方が理解しやすいかもしれません。エンジニア的にわかりやすく言うと、それぞれ以下のようなサービスです。

- Web App : ロードバランサ ＋ クラスタ化された Web サーバ上に、複数のアプリをコンテナ技術により隔離しながら搭載することができる
- SQL DB （上位 SKU の場合） : ログ転送の仕組みにより拠点内で 3 多重化されたデータベース、ログを別データセンタのセカンダリ DB へ非同期転送することで、ほぼリアルタイムに別リージョンへデータを複製できる（日本の東西環境の場合、平常時の遅延は 1 秒以下）

![picture 1](./images/927deb8d243a11d486a58b557341df08c6f82c414da5c58de6230e1e2f042ab1.png)  

本デモでは .NET/SQL アプリを載せるための PaaS インフラとして Web App, SQL DB を利用していますが、これ以外にもコンテナや OSS DB など複数の選択肢があります。載せるアプリに応じて、適切なマネージドインフラを選択してください。

- Web サーバ
  - PaaS, CaaS, IaaS の選択肢があり、さらに複数の選択肢があります。
  - PaaS では AppService (Web App) と Functions、CaaS では AKS (Azure Kubernetes Services) と ACA (Azure Container Apps) が特に重要なサービスです。
  - 詳細は [こちら](https://github.com/Azure/jp-techdocs) の "IaaS/CaaS/PaaS の使い分け"(nakama) を参照してください。
  - ![picture 2](./images/96705459fd984e91c83fe33d1e81a10d8bd882c6dbf40d9b70136f23094a2d87.png)  
- DB サーバ
  - SQL Database 以外の選択肢として、マネージドサービスとしての MySQL, MariaDBm, PostgreSQL があります。
  - Oracle に関しては、OracleDB @ Azure（ODDA）と呼ばれるマネージドサービスが利用できます（簡単に言えば、Azure データセンタ環境内にホストした Exadata を利用できます）。また、OCI (Oracle Cloud Infrastructure) との専用線接続サービスを利用することができます。
  - 詳細は [こちら](https://github.com/Azure/jp-techdocs) の "DB サービスの使い分け"(tfukuha) を参照してください。

本セクションでは、この Web App (App Service) と SQL Databsae を利用して、前セクションと同様の Web-DB システムを構築します。

## サブスクリプションの初期設定

**管理者アカウントで az cli にログインし直します。**

```bash

az account clear
az login

```

引き続き、サブスクリプションの初期設定として、以下を行います。

- プロバイダ登録（App Service を利用できるようにするため、機能の有効化を行います）

```bash

# 対象サブスクリプション
TEMP_TARGET_SUBSCRIPTION_IDS=$SUBSCRIPTION_ID_SPOKE_B

# 登録する Resource Provider と feature を設定
TEMP_RP_NAMES="Microsoft.Web"
TEMP_FEATURE_NAMES="\
"

for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
echo "Initialize subscription... ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"

# RP 有効化
for TEMP_RP_NAME in $TEMP_RP_NAMES; do
echo "Registering ${TEMP_RP_NAME} RP on ${TEMP_SUBSCRIPTION_ID}..."
az provider register --namespace "${TEMP_RP_NAME}"
done #TEMP_RP_NAME

# Feature 有効化
for TEMP_FEATURE_NAME_TEMP in $TEMP_FEATURE_NAMES; do
# 分解して利用
TEMP=(${TEMP_FEATURE_NAME_TEMP//,/ })
TEMP_FEATURE_NAMESPACE=${TEMP[0]}
TEMP_FEATURE_NAME=${TEMP[1]}
az feature register --namespace ${TEMP_FEATURE_NAMESPACE} --name ${TEMP_FEATURE_NAME}
done #TEMP_FEATURE_NAME_TEMP

done #TEMP_SUBSCRIPTION_ID

for TEMP_SUBSCRIPTION_ID in $TEMP_TARGET_SUBSCRIPTION_IDS; do
echo "Waiting initializing subscription... ${TEMP_SUBSCRIPTION_ID}"
az account set -s "${TEMP_SUBSCRIPTION_ID}"

# RP 有効化待ち
for TEMP_RP_NAME in $TEMP_RP_NAMES; do
while [ $(az provider show --namespace "${TEMP_RP_NAME}" --query registrationState -o tsv) != "Registered" ]
do
echo "$(az provider show --namespace "${TEMP_RP_NAME}" --query registrationState -o tsv) on ${TEMP_SUBSCRIPTION_ID} ${TEMP_RP_NAME}..."
sleep 10
done
done #TEMP_RP_NAME

# Feature 有効化待ち
for TEMP_FEATURE_NAME_TEMP in $TEMP_FEATURE_NAMES; do
# 分解して利用
TEMP=(${TEMP_FEATURE_NAME_TEMP//,/ })
TEMP_FEATURE_NAMESPACE=${TEMP[0]}
TEMP_FEATURE_NAME=${TEMP[1]}
while [ $(az feature show --namespace ${TEMP_FEATURE_NAMESPACE} --name ${TEMP_FEATURE_NAME} --query properties.state -o tsv) != "Registered" ]
do
echo "$(az feature show --namespace ${TEMP_FEATURE_NAMESPACE} --name ${TEMP_FEATURE_NAME} --query properties.state -o tsv) ${TEMP_FEATURE_NAMESPACE}/${TEMP_FEATURE_NAME} ..."
sleep 10
done
done #TEMP_FEATURE_NAME_TEMP

done #TEMP_SUBSCRIPTION_ID

```
