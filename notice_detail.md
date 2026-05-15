# 廃止機能 詳細調査報告と改善計画

> 調査日: 2026-05-15  
> 対象: `AzureCAF.LandingZones.Demo` 全 258 MD ファイル  
> 参照元: MS Learn 公式ドキュメント最新版

概要版は [`notice.md`](./notice.md) を参照してください。

---

## 目次

1. [既に対応済みの廃止機能](#1-既に対応済みの廃止機能)
2. [要対応事項 — 優先度【緊急】](#2-要対応事項--優先度緊急)
3. [要対応事項 — 優先度【高】](#3-要対応事項--優先度高)
4. [要対応事項 — 優先度【中】](#4-要対応事項--優先度中)
5. [要対応事項 — 優先度【低・ウォッチ】](#5-要対応事項--優先度低ウォッチ)
6. [改善計画](#6-改善計画)
7. [参考: 主要 MS Learn URL](#7-参考-主要-ms-learn-url)

---

## 1. 既に対応済みの廃止機能

2024/08 の大規模アップデートで対応済み。

| 機能 | 廃止/変更日 | 対応内容 |
|---|---|---|
| Log Analytics Agent (MMA) | 2024-08-31 廃止済 | AMA に移行。MMA ポリシー 4 件を無効化 |
| MMA 自動プロビジョニング | 2024-11-30 廃止済 | `autoProvisioningSettings` を Off に設定 |
| MMA-based FIM | 2024-11 廃止済 | MDE-based FIM (GA: 2024-09-18) に移行 |
| AMA-based FIM (Preview) | 2024-05-30 GA せず廃止 | MDE-FIM を採用 |
| Azure Disk Encryption (ADE) | 2028-09-15 廃止予定 | PMK + DEAH に移行。ADE GC ポリシー 2 件を無効化 |
| Azure Security Benchmark (ASB) → MCSB | 改名済 | MCSB (`1f3afdf9`) に更新済 |
| Security Center → Defender for Cloud | 改名済 | 全ファイルで MDfC 表記 |
| Classic Alerts | 2024 廃止済 | 新 Alert Rules + Processing Rules + Action Groups |
| Qualys ベース脆弱性スキャン | 廃止済 | MdeTvm プロバイダーに変更 |
| kubenet | 2028-03-31 廃止予定 | Azure CNI Overlay + Cilium 使用中 ✅ |

---

## 2. 要対応事項 — 優先度【緊急】

### gpt-35-turbo の退職（AOAI セクション）

**退職日:** 2025-02-13（バージョン 0301/0613）、2025-04-30（16k 0613）  
**現在の状態:** デプロイ済みモデルは動作不可  
**対象ファイル:** `41.Spoke D (AOAI) 社内文書検索 インフラ構築/41_06_PaaS リソース作成.md`

現在の記述では `text-davinci-003 → gpt-35-turbo` という移行経緯を説明しているが、その移行先の gpt-35-turbo も退職済み。

**推奨対応:**
- モデル名を現行の GPT-4 系（`gpt-4o` または `gpt-4.1` シリーズ）に更新
- 実装時は必ず以下ページで最新の GA モデルを確認: https://learn.microsoft.com/azure/ai-services/openai/concepts/model-retirements
- 移行注記の記述も現状に合わせて改訂

---

## 3. 要対応事項 — 優先度【高】

### 3-1. Dependency Agent (DA) の削除

**方針:** Azure 標準機能のみで構成するため DA は削除。VM Insights Map 機能は DA 依存のため、Map も使用しない。

**廃止スケジュール:**

| マイルストーン | 日程 |
|---|---|
| 新 OS バージョン/リージョン追加停止 | 2025-06-30 |
| ポータルからの新規オンボーディング不可 | 2025-09-30 |
| 完全廃止 | 2028-06-30 |

**参照:** https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-maps-retirement

**DA が参照されているファイル:**

| ファイル | 内容 |
|---|---|
| `04.管理基盤の構成設定/04_01_VMInsights用の事前準備.md` (L10, 12, 313–315) | DCR に DependencyAgent の設定を含む |
| `05.仮想マシンのコンプライアンス対応/05_06_★ VM Insights 有効化.md` (L5, 40–43, 52–54, 67–69) | DependencyAgentWindows/Linux の拡張インストール |
| `99.Tips/99_02_az cliの書き方のコツ.md` | DA のインストール例 |

**削除後の VM Insights 構成:** AMA のみ。パフォーマンス監視・ヘルス監視は継続。Map 機能は使用不可（廃止のため問題なし）。

---

### 3-2. Log Analytics Solution の廃止対応

#### "Security" ソリューション

**対象:** `01.初期環境セットアップ/01_01_共用LAW作成.md` (L37)

```bash
az monitor log-analytics solution create --solution-type Security
```

MMA 退職（2024-08-31）に伴い意義が失われている。Defender for Cloud のデータ収集は MDfC 設定（`01_05`）側で管理されており、重複している可能性が高い。  
→ 現在も必要かを検証し、不要であれば削除。

#### SQL 系ソリューション 2 件

**対象:** `04.管理基盤の構成設定/04_02_DefenderForSql用の事前準備.md` (L44–45)

```bash
az monitor log-analytics solution create --solution-type SQLAdvancedThreatProtection
az monitor log-analytics solution create --solution-type SQLVulnerabilityAssessment
```

同ファイルに DCR ベースの設定（`Microsoft-DefenderForSqlAlerts` 等）が既に存在し、重複している可能性が高い。  
→ DCR のみで正常動作することを確認後、LA ソリューション手順を削除。

---

### 3-3. Microsoft.Security REST API バージョンの更新

**対象ファイル:** `01_05_MDfCセットアップ.md`、`99_06_サブスクリプションの一括初期設定.md`

| エンドポイント | 現在 | 推奨 | 場所 |
|---|---|---|---|
| securityContacts | `2020-01-01-preview` | `2023-12-01-preview` | 01_05 (L391), 99_06 (L364) |
| automations | `2019-01-01-preview` | `2023-12-01-preview` | 01_05 (L427) |
| serverVulnerabilityAssessmentsSettings | `2022-01-01-preview` | `2023-05-01` (Stable) | 01_05 (L579) |
| autoProvisioningSettings | `2017-08-01-preview` | 変更不要（唯一のバージョン）| 01_05 (L40) |

**securityContacts 2023-12-01-preview の新機能:** `notificationsSources` コレクション、`isEnabled` プロパティ、アタックパスのリスクレベル設定。

---

### 3-4. Compute・Authorization・Network API バージョンの更新

| エンドポイント | 現在 | 推奨 | 対象ファイル |
|---|---|---|---|
| Microsoft.Compute/virtualMachines | `2015-06-15`（9年前）| `2024-07-01` | `99_04_CompliantVM作成スクリプト.md` |
| Microsoft.Authorization/roleDefinitions | `2018-07-01`（7年前）| `2022-04-01` | `00_05_作業用ユーザアカウントの作成.md` |
| Microsoft.Network/azureFirewalls | `2020-05-01`（5年前）| `2024-05-01` | `02_02_OpsVNETにAzureFirewallを作成.md`, `03_01_HubVNETとAzureFirewallの作成.md` |

**Azure Firewall 追記:** `02_02` と `03_01` に `managementIpConfiguration` が含まれているが、Standard SKU では不要（Premium / 強制トンネリング構成のみ必要）。API バージョン更新と合わせて削除を検討。

---

### 3-5. ContainerLog → ContainerLogV2 移行（期限: 2026-09-30）

**廃止日:** 2026-09-30（残り約 4 ヶ月）  
**対象:** `54.Spoke E (AKS) 監視機能の有効化/` 配下  
**参照:** https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-logs-schema

Container Insights のログスキーマが `ContainerLog` テーブルから `ContainerLogV2` テーブルに移行。ContainerLogV2 はメタデータが豊富でクエリが簡潔、Basic Logs 価格レベルにも対応。

対応内容:
- `ContainerLog` を参照している KQL クエリや手順を `ContainerLogV2` に更新
- ContainerLogV2 への移行手順を追記

---

## 4. 要対応事項 — 優先度【中】

### 4-1. "Azure AD" / "AAD" → "Microsoft Entra ID" への用語変更

Microsoft は 2023 年に Azure Active Directory を **Microsoft Entra ID** に改名。以下 13 ファイルに旧称が残存。

| ファイル | 旧称 |
|---|---|
| `readme.md` (L127, 133, 135, 139, 141) | "Azure AD", "AAD" |
| `00.環境準備/00_01_環境変数設定.md` (L5, 7, 9, 10) | "Azure AD" |
| `01.初期環境セットアップ/01_01_共用LAW作成.md` (L49) | "Azure AD" |
| `02.管理サブスクリプションの作成/02_01_OpsVNET作成.md` (L16) | "Azure AD" |
| `09.Spoke B/09_01_Web アプリセットアップ.md` (L60) | "AAD" |
| `41.Spoke D/41_01_事前準備.md` (L27) | "AAD" |
| `51.Spoke E/51_02_事前準備.md` (L21, 47) | "AAD" |
| `61.Spoke F/61_02_事前準備.md` (L19, 45) | "AAD" |
| `71.DevCenter/71_01_事前準備.md` (L11) | "AAD" |
| `74.DevCenter/74_00_DevBoxの作成について.md` (L15, 19, 20, 21) | "AAD", "Azure AD" |
| `11.Azure Policy/11_10_適用除外(Waiver).md` | "Azure AD" |
| `98.後片付け/98_03_ロール割り当て削除.md` (L40) | "AAD" |
| `98.後片付け/98_04_ユーザの削除.md` (L3) | "Azure AD" |

**変換ルール:** "Azure Active Directory" / "Azure AD" / "AAD" → "Microsoft Entra ID"  
**注意:** Azure CLI の `az ad` サブコマンド名は変更なし（そのまま）。

---

### 4-2. Azure AI Search API バージョン

**対象:** `41_06_PaaS リソース作成.md` (L82, 93)

| 現在 | 推奨 |
|---|---|
| `2021-04-01-Preview` | `2025-09-01` → `2026-04-01` |
| `2022-09-01` | `2025-09-01` → `2026-04-01` |

`2026-04-01` は Agentic Retrieval 関連の破壊的変更を含むため、段階的移行を推奨。  
**参照:** https://learn.microsoft.com/azure/search/search-api-migration

---

### 4-3. Azure Cognitive Services API バージョン

**対象:** `41_06_PaaS リソース作成.md` (L58, 63)

| 現在 | 推奨 |
|---|---|
| `2021-10-01` | `2026-03-01` |

---

### 4-4. Azure Cosmos DB API バージョン

**対象:** `41_12_MID によるアクセス権限設定.md` (L91, 102)

| 現在 | 推奨 |
|---|---|
| `2021-11-15-preview` | `2025-10-15` |

---

### 4-5. ACR authentication-as-arm の設定方向を修正

**対象:** `62_05_コンテナアプリの配置.md` (L52)

```bash
# 現在（セキュリティ上推奨されない）
az acr config authentication-as-arm update -r ${TEMP_ACR_NAME} --status enabled

# 推奨
az acr config authentication-as-arm update -r ${TEMP_ACR_NAME} --status disabled
```

`enabled` は ARM スコープ認証を許可してしまう。`disabled` にすることで ACR スコープの Entra 認証のみ許可され、攻撃面を最小化できる。  
**参照:** https://learn.microsoft.com/azure/container-registry/container-registry-disable-authentication-as-arm

---

### 4-6. Form Recognizer → Azure AI Document Intelligence 用語更新

**対象:** `41_02_メンテ用 VNET 作成.md` (L9)  
2023-07 に改名済み。旧 API エンドポイントは後方互換性あり。用語だけ "Azure AI Document Intelligence" に統一する。

---

### 4-7. MDfC Security Contacts 新スキーマ対応

4-2 の API バージョン更新（`securityContacts` を `2023-12-01-preview` に変更）と連動。新スキーマの `notificationsSources`・`isEnabled` フィールドを活用するようコマンドを改訂する。

---

### 4-8. ADE ドキュメント構造見直し

**対象:** `02_06_（参考）ディスクの暗号化.md`

廃止注記は追加済みだが、ページ構造は ADE 中心のまま。DEAH を主要ガイダンスに昇格し、ADE を「廃止技術・参考情報」として末尾に移動する。廃止後の影響（再起動後にアンロック失敗）を警告として強調する。

---

### 4-9. OperationalInsights API バージョン確認

`01_01_共用LAW作成.md` での `Microsoft.OperationalInsights/workspaces` の API バージョンを確認。最新安定版は `2023-09-01`。古い場合は更新する。

---

## 5. 要対応事項 — 優先度【低・ウォッチ】

| # | 対象 | 内容 |
|---|---|---|
| 1 | DevCenter API バージョン | `74_09` で `2023-06-01-preview` / `2023-04-01` を使用中。最新安定版（2024 系）への更新を検討 |
| 2 | Private DNS Zone API | 複数ファイルで `2020-06-01`（6年前）。動作するが最新版（`2024-06-01`）への更新を検討 |
| 3 | "old_MDfS" ファイアウォールルール | `01_06` に `old_MDfS(MDE)` 等が残存。要否確認後に削除 |
| 4 | Container Insights 監視構成 | Container Insights + Managed Prometheus を並存使用中。廃止予定はないが将来方針を注記 |
| 5 | Azure Policy Add-on for AKS | `--addons azure-policy` を使用。廃止予定なし（v1.16.1 最新）。継続モニタリング |

---

## 6. 改善計画

総推定工数: **約 7 日**

| フェーズ | 内容 | 工数 | 期限 |
|---|---|---|---|
| 1 | AOAI モデル更新（gpt-35-turbo → gpt-4o/4.1 系） | 0.5日 | 即時 |
| 2 | DA (Dependency Agent) 削除 | 1日 | — |
| 3 | Log Analytics Solution 廃止対応 | 0.5日 | — |
| 4 | Security API バージョン更新 (securityContacts, automations 等) | 1日 | — |
| 5 | Compute・Authorization・Network API バージョン更新 | 1日 | — |
| 6 | ContainerLog → ContainerLogV2 移行 | 0.5日 | **2026-09-30** |
| 7 | 用語更新（Entra ID、Form Recognizer、ADE 構造）| 1日 | — |
| 8 | AOAI 系 API バージョン更新（AI Search、Cosmos DB、Cognitive Services） | 0.5日 | — |
| 9 | ACR 設定修正・old_MDfS 整理・DevCenter API 確認 | 0.5日 | — |

---

## 7. 参考: 主要 MS Learn URL

| 機能 | URL |
|---|---|
| Log Analytics Agent 移行 | https://learn.microsoft.com/azure/azure-monitor/agents/azure-monitor-agent-migration |
| VM Insights Map 廃止 | https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-maps-retirement |
| Azure Disk Encryption 移行 | https://learn.microsoft.com/azure/virtual-machines/disk-encryption-migrate |
| MDfC MMA 廃止準備 | https://learn.microsoft.com/azure/defender-for-cloud/prepare-deprecation-log-analytics-mma-agent |
| Azure OpenAI モデル退職 | https://learn.microsoft.com/azure/ai-services/openai/concepts/model-retirements |
| Container Insights ログスキーマ | https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-logs-schema |
| ACR ARM 認証の無効化 | https://learn.microsoft.com/azure/container-registry/container-registry-disable-authentication-as-arm |
| Azure AI Search API 移行 | https://learn.microsoft.com/azure/search/search-api-migration |
| AKS ネットワーク概要 | https://learn.microsoft.com/azure/aks/concepts-network-cni-overview |
| Azure Policy for Kubernetes | https://learn.microsoft.com/azure/governance/policy/concepts/policy-for-kubernetes |
| DCR 概要 | https://learn.microsoft.com/azure/azure-monitor/data-collection/data-collection-rule-overview |
| OperationalInsights ARM | https://learn.microsoft.com/azure/templates/microsoft.operationalinsights/workspaces |

---

*作成: 2026-05-15*
