# Azure Firewall の経路開放

今回は vm-mtn-XXX 端末上でアプリとコンテナのビルドを行いますが、そのためにインターネットからいくつかのファイルの入手が必要になります。以下のコマンドを実行して ops 上の Azure Firewall に穴をあけ、vm-mtn-XXX 上で以下の作業ができるようにします。

※ 下記では作業を容易にするため、すべての FQDN に対して穴あけをしています。作業後に Azure Firewall のログを確認して FQDN を絞るなどの工夫を行ってください。

```bash

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ハブサブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_MGMT}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# 操作する Firewall Policy
TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
TEMP_FWP_NAME="fw-ops-${TEMP_LOCATION_PREFIX}-fwp"
# 通信元
TEMP_IP_PREFIX=${IP_SPOKE_D_PREFIXS[$i]}
TEMP_SUBNET_DEFAULT="${TEMP_IP_PREFIX}.128.0/24"

az network firewall policy rule-collection-group collection add-filter-collection \
--resource-group ${TEMP_RG_NAME} --policy-name ${TEMP_FWP_NAME} --rcg-name "DefaultApplicationRuleCollectionGroup" \
--name "Spoke D AOAI" --rule-type ApplicationRule --collection-priority 50400 --action Allow \
--rule-name "All Web Sites" \
--target-fqdns "*" \
--source-addresses ${TEMP_SUBNET_DEFAULT} --protocols Https=443 Http=80

done # TEMP_LOCATION

```
