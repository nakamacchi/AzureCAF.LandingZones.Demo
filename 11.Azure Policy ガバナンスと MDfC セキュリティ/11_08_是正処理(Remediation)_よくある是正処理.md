# 是正処理 (Remediation) : よくある是正処理

是正可能なものは適宜修正を行う必要があります。ここでは、以下の是正処理を行います。

- セキュリティパッチの適用指示
  - 未適用のセキュリティパッチがある場合、MDE がこれを検出し、推奨事項（非準拠事項）として報告してきます。セキュリティパッチは速やかに適用するようにしてください。
  - セキュリティパッチの適用は、UMC (Update Management Center) と呼ばれる機能を使って行うことができます。仮想マシンに標準インストールされている VM ゲストエージェントの機能が利用されるため、特に追加のエージェントのインストールは不要です。

```bash

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

#####################################
# セキュリティパッチの適用指示

# 運用管理の日常作業担当者のアカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_mgmt_ops"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_mgmt_ops@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# パッチのアセスメント
#for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_IDS; do
#az account set -s $TEMP_SUBSCRIPTION_ID
#for TEMP_VM_ID in $(az vm list --query [].id -o tsv); do
#echo ${TEMP_VM_ID}
#az rest --method post --url "${TEMP_VM_ID}/assessPatches?api-version=2021-03-01"
#done
#done
 
# パッチの適用
for TEMP_SUBSCRIPTION_ID in $SUBSCRIPTION_IDS; do
az account set -s $TEMP_SUBSCRIPTION_ID
for TEMP_VM_ID in $(az vm list --query [].id -o tsv); do
echo ${TEMP_VM_ID}
az rest --method post --url "${TEMP_VM_ID}/installPatches?api-version=2022-03-01" --body "{ \"maximumDuration\": \"PT4H\", \"rebootSetting\": \"IfRequired\", \"windowsParameters\": { \"classificationsToInclude\": [ \"Critical\", \"Security\", \"UpdateRollUp\", \"FeaturePack\", \"ServicePack\", \"Definition\", \"Tools\", \"Updates\" ] }, \"linuxParameters\": { \"classificationsToInclude\": [ \"Critical\", \"Security\", \"Other\" ] } }"
done
done

# 業務システム統制チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_gov_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi
TEMP_SUBSCRIPTION_IDS=$SUBSCRIPTION_IDS

```
