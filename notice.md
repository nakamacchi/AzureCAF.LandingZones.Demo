# 未対応の廃止・変更機能について

> 最終確認日: 2026-05-15  
> 状況: 工数確保ができておらず、以下の対応が未実施のまま残っています。

本リポジトリは 2024/08 に主要な廃止機能（MMA / ADE / MMA-based FIM 等）の対応を実施しましたが、  
その後も Azure 側での変更が続いており、現時点で対応できていない箇所があります。

---

## ⚠️ 既に動作しない箇所

| # | 対象 | 内容 | 影響ファイル |
|---|---|---|---|
| 1 | **Azure OpenAI gpt-35-turbo** | 2025-02-13 に退職済み。手順通りに進めてもモデルのデプロイが失敗します | `41_06_PaaS リソース作成.md` |

---

## 🔴 期限あり・早急に対応が必要

| # | 対象 | 期限 | 内容 | 影響ファイル |
|---|---|---|---|---|
| 2 | **ContainerLog → ContainerLogV2** | **2026-09-30** | Container Insights のログテーブルが廃止。この期限を過ぎると AKS のログクエリが動作しなくなります | `54.Spoke E (AKS)` 配下 |

---

## 🟠 廃止予定・対応推奨

| # | 対象 | 廃止予定日 | 内容 | 影響ファイル |
|---|---|---|---|---|
| 3 | **Dependency Agent (DA)** | 2028-06-30 | VM Insights の Map 機能依存。Azure 標準機能のみで構成する方針のため削除予定 | `04_01_VMInsights用の事前準備.md`, `05_06_★ VM Insights 有効化.md`, `99_02_az cliの書き方のコツ.md` |
| 4 | **Log Analytics Solution "Security"** | 移行中 | MMA 退職に伴い意義が薄れた。DCR + AMA / エージェントレスへの移行が推奨 | `01_01_共用LAW作成.md` |
| 5 | **Log Analytics Solution SQL 系 2 件** | 移行中 | DCR ベースの設定と重複している可能性あり | `04_02_DefenderForSql用の事前準備.md` |

---

## 🟡 API バージョンの陳腐化

手順は動作しますが、使用している API バージョンが古くなっています。

| # | リソース | 現在 | 推奨 | 対象ファイル |
|---|---|---|---|---|
| 6 | Microsoft.Security/securityContacts | 2020-01-01-preview | 2023-12-01-preview | `01_05_MDfCセットアップ.md`, `99_06_サブスクリプションの一括初期設定.md` |
| 7 | Microsoft.Security/automations | 2019-01-01-preview | 2023-12-01-preview | `01_05_MDfCセットアップ.md` |
| 8 | Microsoft.Security/serverVulnerabilityAssessmentsSettings | 2022-01-01-preview | 2023-05-01 | `01_05_MDfCセットアップ.md` |
| 9 | Microsoft.Compute/virtualMachines | 2015-06-15 | 2024-07-01 | `99_04_CompliantVM作成スクリプト.md` |
| 10 | Microsoft.Authorization/roleDefinitions | 2018-07-01 | 2022-04-01 | `00_05_作業用ユーザアカウントの作成.md` |
| 11 | Microsoft.Network/azureFirewalls | 2020-05-01 | 2024-05-01 | `02_02_OpsVNETにAzureFirewallを作成.md`, `03_01_HubVNETとAzureFirewallの作成.md` |
| 12 | Azure AI Search | 2021-04-01-Preview / 2022-09-01 | 2025-09-01 以降 | `41_06_PaaS リソース作成.md` |
| 13 | Azure Cognitive Services | 2021-10-01 | 2026-03-01 | `41_06_PaaS リソース作成.md` |
| 14 | Azure Cosmos DB | 2021-11-15-preview | 2025-10-15 | `41_12_MID によるアクセス権限設定.md` |

---

## 🔵 設定・表記の更新

| # | 対象 | 内容 | 影響ファイル |
|---|---|---|---|
| 15 | **"Azure AD" / "AAD" 用語** | Microsoft Entra ID に改名済み (2023)。18 ファイル以上に旧称が残存 | 複数ファイル（詳細は下記）|
| 16 | **ACR authentication-as-arm** | `--status enabled` になっているが、セキュリティ推奨は `--status disabled` | `62_05_コンテナアプリの配置.md` |
| 17 | **Form Recognizer 用語** | Azure AI Document Intelligence に改名済み (2023-07) | `41_02_メンテ用 VNET 作成.md` |
| 18 | **Azure Firewall managementIpConfiguration** | Standard SKU では不要。API 更新と合わせて見直し | `02_02`, `03_01` |
| 19 | **ADE ページ構造** | 廃止注記はあるが DEAH を主要ガイダンスに格上げしたい | `02_06_（参考）ディスクの暗号化.md` |
| 20 | **"old_MDfS" ファイアウォールルール** | "old" 接頭辞のルール群が残存。要否確認後に削除 | `01_06_共通フィルタルール（AzureFirewallポリシー）の作成.md` |

### "Azure AD" / "AAD" が残っているファイル一覧

`readme.md`, `00_01_環境変数設定.md`, `01_01_共用LAW作成.md`, `02_01_OpsVNET作成.md`,  
`09_01_Web アプリセットアップ.md`, `41_01_事前準備.md`, `51_02_事前準備.md`,  
`61_02_事前準備.md`, `71_01_事前準備.md`, `74_00_DevBoxの作成について.md`,  
`11_10_適用除外(Exemption)_よくある免除項目(Waiver).md`, `98_03_ロール割り当て・カスタムロールの全削除.md`, `98_04_ユーザの削除.md`

---

## ✅ 対応済み（参考）

2024/08 のアップデートで対応済みの主要廃止機能です。

| 機能 | 対応内容 |
|---|---|
| Log Analytics Agent (MMA) — 2024-08-31 廃止 | AMA に移行済み。関連ポリシー 4 件を無効化 |
| MMA-based FIM — 2024-11 廃止 | MDE-based FIM (GA: 2024-09-18) に移行済み |
| AMA-based FIM (Preview) — GA せず廃止 | MDE-FIM を採用済み |
| Azure Disk Encryption (ADE) — 2028-09-15 廃止予定 | PMK + DEAH に移行済み |
| Azure Security Benchmark → MCSB | MCSB に更新済み |
| Classic Alerts — 2024 廃止 | 新 Alert Rules に移行済み |
| Qualys 脆弱性スキャン | MDE TVM に移行済み |
| kubenet — 2028-03-31 廃止予定 | Azure CNI Overlay + Cilium 使用中 |

---

詳細な調査報告・改善計画は [`notice_detail.md`](./notice_detail.md) を参照してください。
