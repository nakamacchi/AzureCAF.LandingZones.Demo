## あとまわし

- MID による ACR アクセス（az コマンドを使わないやり方）
- 以下のような流れらしいがドキュメントがなく正確なパラメータ指定が不明（特にトークンの取り出し方）。以下の URL の通りだが非常に大変。
https://shuanglu1993.medium.com/access-the-azure-container-registry-using-azure-managed-identity-programatically-d203ca17170b

```

TEMP_ACR_NAME="acrspoked${UNIQUE_SUFFIX}${TEMP_LOCATION_PREFIX}"
TEMP_ACR_RES_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_D}/resourceGroups/rg-spoked-${TEMP_LOCATION_PREFIX}/providers/Microsoft.ContainerRegistry/registries/${TEMP_ACR_NAME}"
# または
# curl -H Metadata:true "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/"
cat <<EOF
TEMP_RESULT=\$(curl -H Metadata:true "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/")
TEMP_ACCESS_TOKEN=\${TEMP_RESULT#*\"access_token\":\"}
TEMP_ACCESS_TOKEN=\${TEMP_ACCESS_TOKEN%%\"*}
TEMP_CLIENT_ID=\${TEMP_RESULT#*\"client_id\":\"}
TEMP_CLIENT_ID=\${TEMP_CLIENT_ID%%\"*}
echo \$TEMP_ACCESS_TOKEN | sudo docker login ${TEMP_ACR_NAME}.azurecr.io -u 00000000-0000-0000-0000-000000000000 --password-stdin
EOF

```

```bash

#TEMP_CAE_NAME="cae-spoked-${TEMP_LOCATION_PREFIX}"
#TEMP_CA_NAME="ca-spoked-${TEMP_LOCATION_PREFIX}"
# ACA から ACR への MID によるコンテナ pull がうまく動作しないため、管理者アカウントでアクセスする
# az containerapp registry set --resource-group "${TEMP_RG_NAME}" --name "${TEMP_CA_NAME}" --server ${TEMP_ACR_NAME}.azurecr.io --identity system
# az containerapp registry set --resource-group "${TEMP_RG_NAME}" --name "${TEMP_CA_NAME}" --server ${TEMP_ACR_NAME}.azurecr.io --username ${TEMP_ACR_USERNAME} --password ${TEMP_ACR_PASSWORD}

```
