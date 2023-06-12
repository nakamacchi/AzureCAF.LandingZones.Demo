# ポリシー適用除外 : （参考）ポリシーのdisabled化

## 古いチェックポリシーの無効化

下記の EDR 関連ポリシーは MDfC 側からデータが報告されないため、Azure Policy 側では not compliant 扱いになってしまいます。このため、割り当て時のパラメータを調整（Disabled 化）するとよいでしょう。

| MDfC 推奨事項 （日本語） | MDfC 推奨事項 （英語） | ID (Microsoft.Security/assessments) | Azure Policy （日本語） | Azure Policy （英語語） | ID |
| --- | --- | --- | --- | --- | --- |
| エンドポイント保護をマシンにインストールする必要がある | Endpoint protection should be installed on machines | 4fb67663-9ab9-475d-b026-8c544cced439 | エンドポイント保護をお使いのマシンにインストールする必要がある | Endpoint protection should be installed on your machines | 1f7c564c-0a90-4d44-b7e1-9d456cffaee8 |
| マシンのエンドポイント保護の正常性の問題を解決する必要がある | Endpoint protection health issues on machines should be resolved | 37a3689a-818e-4a0e-82ac-b1392b9bb000 | Endpoint Protection の正常性の問題を、お使いのコンピューターで解決する必要があります | Endpoint protection health issues should be resolved on your machines | 8e42c1f2-a2ab-49bc-994a-12bcd0dc4ac2 |

内容としては以下でカバーされています。(EPP = EDR + Update Management)

| MDfC 推奨事項 （日本語） | MDfC 推奨事項 （英語） | ID (Microsoft.Security/assessments) | Azure Policy （日本語） | Azure Policy （英語語） | ID |
| --- | --- | --- | --- | --- | --- |
| マシンには脆弱性評価ソリューションが必要 | Machines should have a vulnerability assessment solution | ffff0522-1e88-47fc-8382-2a80ba848f5d | 脆弱性評価ソリューションを仮想マシンで有効にする必要があります | A vulnerability assessment solution should be enabled on your virtual machines | 501541f7-f7e7-4cd6-868c-4190fdad3ac9 |
| マシンでは、脆弱性の検出結果が解決されている必要がある | Machines should have vulnerability findings resolved | 1195afff-c881-495e-9bc5-1486211ae03f |   |   |   |
| システム更新プログラムがマシンにインストールされている必要がある (Update Management センターを利用) | System updates should be installed on your machines (powered by Update management center) | e1145ab1-eb4f-43d8-911b-36ddf771d13f | [プレビュー]: システム更新プログラムがマシンにインストールされている必要がある (更新センターを利用) | [Preview]: System updates should be installed on your machines (powered by Update Center) | f85bf3e0-d513-442e-89c3-1784ad63382b |
| 不足しているシステム更新プログラムを定期的に確認するようにマシンを構成する必要がある | Machines should be configured to periodically check for missing system updates | 90386950-71ca-4357-a12e-486d1679427c | [プレビュー]: 不足しているシステム更新プログラムを定期的に確認するようにマシンを構成する必要がある | [Preview]: Machines should be configured to periodically check for missing system updates | bd876905-5b84-4f73-ab2d-2e7a7c4568d9 |

ASB 内部でのパラメータ名は以下の 2 つです。既定値では下方互換性のため AuditIfNotExist になっていますが、これらを Disabled にしていただくことで Azure Policy 側でのチェックを無効化（MDfC からのデータ取り込みを無視）できます。（カッコ内は日本語名）

- installEndpointProtectionMonitoringEffect （エンドポイント保護をお使いのマシンにインストールする必要がある）
- endpointProtectionHealthIssuesMonitoringEffect（Endpoint Protection の正常性の問題を、お使いのコンピューターで解決する必要があります）

![picture 1](./images/34c834ff4ab98844865f4a8bf766998c63b14e2bcd0317c1cbdbfb100a96919d.png)  

![picture 2](./images/dcc273be8dac0cfddbb1977fde093580880045ae12c7ab12107373ed1cdbcc8c.png)  
