# Azure Firewall の経路開放

今回は vm-mtn-XXX 端末上でアプリとコンテナのビルドを行いますが、そのためにインターネットからいくつかのファイルの入手が必要になります。以下のコマンドを実行して ops 上の Azure Firewall に穴をあけ、vm-mtn-XXX 上で以下の作業ができるようにします。

- WSL のインストール（Ubuntu のインストールも含む）
- サンプルアプリのダウンロード
- NuGet パッケージのダウンロード（コンテナビルド中に行われます）

※ 下記で開放する FQDN の中には *.githubusercontent.com など、安全とは言えないものが含まれています。今回はこれらの FQDN を開放していますが、よりセキュアな環境にしたい場合には、コンテナをオンプレミスなどでビルドし、作成されたイメージだけを ACR にアップロードして使う、などの工夫をしてください。

```bash

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password '${SP_PWDS[${TEMP_SP_NAME}]}' --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ハブサブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_MGMT}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

# 操作する Firewall Policy
TEMP_RG_NAME="rg-ops-${TEMP_LOCATION_PREFIX}"
TEMP_FWP_NAME="fw-ops-${TEMP_LOCATION_PREFIX}-fwp"
# 通信元
TEMP_IP_PREFIX=${IP_SPOKE_F_PREFIXS[$i]}
TEMP_SUBNET_DEFAULT="${TEMP_IP_PREFIX}.128.0/24"

az network firewall policy rule-collection-group collection add-filter-collection \
--resource-group ${TEMP_RG_NAME} --policy-name ${TEMP_FWP_NAME} --rcg-name "DefaultApplicationRuleCollectionGroup" \
--name "ResourcesForContainerBuild" --rule-type ApplicationRule --collection-priority 50600 --action Allow \
--rule-name "GitHub" \
--target-fqdns "github.com" "*.githubusercontent.com" "*.github.com" \
--source-addresses ${TEMP_SUBNET_DEFAULT} --protocols Https=443

az network firewall policy rule-collection-group collection rule add \
--resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
--collection-name "ResourcesForContainerBuild" --rule-type ApplicationRule \
--name "WslStore" \
--target-fqdns "wslstorestorage.blob.core.windows.net" \
--source-addresses ${TEMP_SUBNET_DEFAULT} --protocols Https=443

az network firewall policy rule-collection-group collection rule add \
--resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
--collection-name "ResourcesForContainerBuild" --rule-type ApplicationRule \
--name "Docker" \
--target-fqdns "download.docker.com" \
--source-addresses ${TEMP_SUBNET_DEFAULT} --protocols Https=443

az network firewall policy rule-collection-group collection rule add \
--resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
--collection-name "ResourcesForContainerBuild" --rule-type ApplicationRule \
--name "MCR" \
--target-fqdns "mcr.microsoft.com" "*.data.mcr.microsoft.com" \
--source-addresses ${TEMP_SUBNET_DEFAULT} --protocols Https=443

az network firewall policy rule-collection-group collection rule add \
--resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
--collection-name "ResourcesForContainerBuild" --rule-type ApplicationRule \
--name "NuGet" \
--target-fqdns "api.nuget.org" \
--source-addresses ${TEMP_SUBNET_DEFAULT} --protocols Https=443

az network firewall policy rule-collection-group collection rule add \
--resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
--collection-name "ResourcesForContainerBuild" --rule-type ApplicationRule \
--name "Microsoft Download" \
--target-fqdns "aka.ms" "go.microsoft.com" "download.microsoft.com" "learn.microsoft.com" \
--source-addresses ${TEMP_SUBNET_DEFAULT} --protocols Https=443

az network firewall policy rule-collection-group collection rule add \
--resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
--collection-name "ResourcesForContainerBuild" --rule-type ApplicationRule \
--name "Alpine Linux Download" \
--target-fqdns "dl-cdn.alpinelinux.org" \
--source-addresses ${TEMP_SUBNET_DEFAULT} --protocols Https=443

az network firewall policy rule-collection-group collection rule add \
--resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "DefaultApplicationRuleCollectionGroup" \
--collection-name "ResourcesForContainerBuild" --rule-type ApplicationRule \
--name "az cli download" \
--target-fqdns "azurecliprod.blob.core.windows.net" \
--source-addresses ${TEMP_SUBNET_DEFAULT} --protocols Https=443

done # TEMP_LOCATION

```
