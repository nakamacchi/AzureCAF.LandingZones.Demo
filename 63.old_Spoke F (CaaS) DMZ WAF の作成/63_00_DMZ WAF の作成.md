# DMZ WAF の作成

Container Apps にインターネットからアクセスできるようにするため、DMZ WAF を作成します。

- Container Apps (CA) をホストしている Container Apps Environment (CAE、物理的な実態は AKS による k8s クラスタ) は、アプリをそのまま外部へ公開する機能を持っています。しかし Web アプリの場合にはインターネットへ直接公開するよりも、WAF を間に挟むことが推奨されますので、今回も CAE の外側に WAF を立てるように構成します。
- WAF から CA へのアクセスに関しては、VNET ピアリングを利用します。他のサンプル（Spoke B PaaS サンプル）ではプライベートエンドポイントを利用することよって隔離性を高めていますが、CA/CAE はプライベートエンドポイントをサポートしていません。このため、VNET ピアリングにより WAF から CA へのアクセスを行います。

![Picture 0](../61.Spoke%20F%20(CaaS)%20業務サブスクリプションの作成/images/64361d769516d8ddabb5859196d891486b10f1ef549e28c6600f3674ea8a215d.png)

- なお、本来であればこの Application Gateway では以下の 2 つの機能を有効化すべきですが、いずれも非常に高額なリソースであるため、今回は割愛します。
  - WAF 機能 (WAF SKU)
  - DDoS Protection Standard
- また、コストの関係で Basic SKU を利用します。2024/07 時点では preview のため、以下のスクリプトで機能を有効化してください。

```bash

# 業務システム B チーム／① 初期構築の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_spokef_dev"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password ${SP_PWDS[${TEMP_SP_NAME}]} --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_spokef_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

TEMP_TARGET_SUBSCRIPTION_IDS=$SUBSCRIPTION_ID_SPOKE_F

# 登録する Resource Provider と feature を設定
TEMP_RP_NAMES=""
TEMP_FEATURE_NAMES="\
Microsoft.Network,AllowApplicationGatewayBasicSku \
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
