# DevBox の作成

以上の作業が終わったら、開発プロジェクトメンバーのアカウントでセルフサービスポータルにアクセスし、開発ボックスを要求してください。

## ユーザアカウントについて

以下のスクリプトで、ログインに利用するユーザアカウントを取得しておいてください。

```bash

cat <<EOF
user_projectx_user1@${PRIMARY_DOMAIN_NAME}
user_projectx_user2@${PRIMARY_DOMAIN_NAME}
user_projectx_user3@${PRIMARY_DOMAIN_NAME}
user_projectx_admin@${PRIMARY_DOMAIN_NAME}
user_projecty_user1@${PRIMARY_DOMAIN_NAME}
user_projecty_user2@${PRIMARY_DOMAIN_NAME}
user_projecty_user3@${PRIMARY_DOMAIN_NAME}
user_projecty_admin@${PRIMARY_DOMAIN_NAME}
EOF

```

