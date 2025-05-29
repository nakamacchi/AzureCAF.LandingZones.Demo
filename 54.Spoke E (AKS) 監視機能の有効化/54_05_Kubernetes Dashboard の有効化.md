# Kubernetes Dashboard の有効化

以下の手順で k8s Dashboard を利用することができます。ユーザについては https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md の手順に沿って作成します。

まず、Docker レジストリから k8s Dashboard のイメージを取得できるように穴を空けます。

```bash

# NW 構成管理チーム／③ 構成変更の作業アカウントに切り替え
if ${FLAG_USE_SOD}; then if ${FLAG_USE_SOD_SP}; then TEMP_SP_NAME="sp_nw_change"; az login --service-principal --username ${SP_APP_IDS[${TEMP_SP_NAME}]} --password "${SP_PWDS[${TEMP_SP_NAME}]}" --tenant ${PRIMARY_DOMAIN_NAME} --allow-no-subscriptions; else az account clear; az login -u "user_nw_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}"; fi; fi

# ハブサブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_HUB}"

for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-hub-${TEMP_LOCATION_PREFIX}"
TEMP_FWP_NAME="fw-hub-${TEMP_LOCATION_PREFIX}-fwp"

TEMP_SPOKE_ADDRESS="${IP_SPOKE_E_PREFIXS[$i]}.0.0/16"

az network firewall policy rule-collection-group collection rule add \
--resource-group "${TEMP_RG_NAME}" --policy-name "${TEMP_FWP_NAME}" --rcg-name "AdditionalApplicationRuleCollectionGroup" \
--collection-name "Spoke E AKS" --rule-type ApplicationRule \
--name "Docker" \
--target-fqdns "download.docker.com" "registry-1.docker.io" "docker.io" "auth.docker.io" "production.cloudflare.docker.com" \
--source-addresses ${TEMP_SPOKE_ADDRESS} --protocols Https=443

done # TEMP_LOCATION

```

**vm-mtn-xxx** で kubectl を使い、以下を実行して k8s Dashboard を立ち上げます。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml

kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

kubectl -n kubernetes-dashboard create token admin-user
```

上記で表示されるトークンをコピーしておきます。

```bash
kubectl proxy
```

上記を実行した後、以下の URL にアクセスすると k8s Dashboard が使えるようになります。ノードプールの CPU・メモリ要求状況などを簡単に調べることができます。

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

![picture 0](./images/6bfade4de0aafcc0f2f459bcf2c789056ede9f5ca8c4976d40bb28791561f4e2.png)  

うまく起動しない場合には、以下で調査を進めてください。よくある原因は以下です。

- ノードリソース（CPU/Memory）が不足している
- イメージが pull できない
- ネットワーク関連（CNI など）の問題

```bash
# 1. Pod の状態を確認
kubectl -n kubernetes-dashboard get pods

# 2. Pod のイベントとログを確認
kubectl -n kubernetes-dashboard describe pod <POD_NAME>
kubectl -n kubernetes-dashboard logs <POD_NAME>

# 3. サービスとエンドポイントの確認
kubectl -n kubernetes-dashboard get svc
kubectl -n kubernetes-dashboard get endpoints

```
