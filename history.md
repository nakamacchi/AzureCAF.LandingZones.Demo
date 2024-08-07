# 主な更新ポイントについて

※ 2024/08 の更新以前のコンテンツを参照・確認したい場合には、[こちら](https://github.com/nakamacchi/AzureCAF.LandingZones.Demo/tree/6a63ddf8df4ce20d9de05d3b29a90ce96e5694ae)をご確認ください。

## 2024/08 更新

- 主な変更点
  - サービスプリンシパルによる権限分掌作業への対応
    - MFA 強制環境ではユーザアカウントを使った作業アカウント切り替えが難しいため
    - FLAG_USE_SP = true を指定して利用できるように全体を修正
  - 仮想マシンの管理方式の見直し
    - ADE (BitLocker) ではなく DiskEncryptionAtHost によるディスク暗号化方式に変更 (鍵管理が不要で扱いがラク)
    - セキュリティエージェントの統廃合に伴う拡張機能の見直し
      - ASA (Azure Security Agent) が MDE に統合され、不要になりました
      - Guest Attestation 拡張機能を追加しました（Trusted Launch の不正改ざんなどを検出するためのモジュール）
      - SQLVA, SQLATP エージェントが AMA に対応したため、MMA を削除しました
  - 仮想マシン SKU の見直し
    - Standard D2v2 から Standard_D2ads_v5 へ変更
      - Trusted Launch を有効化した状態で WSL2 を使えるようにするための変更
      - D2ads_v5 が利用できる環境が増えたため、SKU を変更しました
  - Azure Firewall のルールの見直し
    - MDE 通信ルールの見直し
      - Streamlined URL と呼ばれる仕組みにより FQDN が簡素化されました
      - 古いクライアントをサポートする目的で、以前の設定も残してあります
  - 利用する SKU の見直し
    - Application Gateway を Standard V2 から Basic V2 へ変更（コストが 10 倍近く違うため）
  - Azure Policy の見直し
    - 適用除外項目などを見直し
    - 無効化するポリシーの見直し
      - 主に MMA の廃止に伴うポリシーを無効化しています
  - ACA サンプル（Spoke F）の改善
    - 複数のアプリを配置するように変更しました
    - アプリケーションを ASP.NET Core MVC アプリから ASP.NET Core Blazor アプリに変更しました
  - Entra ID 認証の利用
    - ACA サンプル（Spoke F）でのリソースアクセス（SQL DB, ACR）を、Managed ID による Entra ID 認証方式に切り替えました
      - 管理 VM 及び ACA 上のアプリの MID への権限付与のため、user_spokef_dev に付与する権限も併せて見直しています
      - これに伴い、SQL DB のセキュリティ構成違反が是正されたため、除外 (exemption) も見直しています
    - なお Spoke A, B のサンプルについては SQL Server 認証方式のまま変更していません。
      - これは利用しているサンプルアプリが .NET Framework 4 ベースで、Managed ID による Entra ID 認証のためにはアプリ書き換えが必要になるためです。古いコードのアプリを IaaS などに配置する場合にはよくあるパターンだと考えられるため、そのままにしてあります。
  - AKS サンプル (Spoke E) の追加
    - 以前公開していた FgCF AKS リファレンスアーキテクチャを最新化・見直しする形で、この基盤上に再実装しました
    - 以前のリファレンスアーキテクチャと同じ点
      - 閉域構成
      - 保守作業用の VM を用意して使う
    - 以前のリファレンスアーキテクチャから変更・強化した点
      - 本番系／運用系を分離
      - サンプルアプリを変更 (Blazor United, Server アプリに変更)
      - Ingress コントローラを利用 (approuting アドオンを利用)、Private Link Service の利用を廃止
      - ネットワークプラグインを Azure CNI Powered by Cilium に変更し、Pod IP アドレスを仮想化
      - システムノードプールとユーザノードプールを分離（4 台で構成）
      - Managed ID を利用して SQL DB や ACR へアクセス（システム MID とワークロード MID も分離）
      - 運用監視基盤として Managed Prometheus, Managed Grafana を追加
    - 未実装・未対応
      - Azure Policy によるコンプライアンス対応
      - ドキュメンテーション
  - FISC 安全対策基準への準拠
    - FISC 安全対策基準と照らし合わせた場合に問題となる点の有無を確認
    - 本サンプルで未実装（スコープ外）の部分として、利用元 IP アドレス制限があります。こちらは Entra ID の条件付きアクセスにより設定してください。
- 細かい変更点
  - 利用制限の確認処理の追加
- 未対応部分
  - Step 71～74 DevCenter 部分については、サービスプリンシパルによる権限分掌作業への対応が済んでいません。
