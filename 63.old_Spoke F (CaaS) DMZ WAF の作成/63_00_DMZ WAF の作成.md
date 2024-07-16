# DMZ WAF の作成

Container Apps にインターネットからアクセスできるようにするため、DMZ WAF を作成します。

- Container Apps (CA) をホストしている Container Apps Environment (CAE、物理的な実態は AKS による k8s クラスタ) は、アプリをそのまま外部へ公開する機能を持っています。しかし Web アプリの場合にはインターネットへ直接公開するよりも、WAF を間に挟むことが推奨されますので、今回も CAE の外側に WAF を立てるように構成します。
- WAF から CA へのアクセスに関しては、VNET ピアリングを利用します。他のサンプル（Spoke B PaaS サンプル）ではプライベートエンドポイントを利用することよって隔離性を高めていますが、CA/CAE はプライベートエンドポイントをサポートしていません。このため、VNET ピアリングにより WAF から CA へのアクセスを行います。

![Picture 0](../61.Spoke%20F%20(CaaS)%20業務サブスクリプションの作成/images/64361d769516d8ddabb5859196d891486b10f1ef549e28c6600f3674ea8a215d.png)

- なお、本来であればこの Application Gateway では以下の 2 つの機能を有効化すべきですが、いずれも非常に高額なリソースであるため、今回は割愛します。
  - WAF 機能 (WAF SKU)
  - DDoS Protection Standard
