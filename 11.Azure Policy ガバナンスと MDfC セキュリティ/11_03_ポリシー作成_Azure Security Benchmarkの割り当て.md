# ポリシー作成 : Azure Security Benchmark の割り当て

## （参考） ASB 適用に関する注意点

- ASB は比較的頻繁にバージョンアップされており、下記のスクリプトではその時点での最新版が適用される。このため、後の作業で実施する適用免除などについては過不足が生じる場合があり、適宜修正が必要。
- 更新履歴は下記を参照。
  - https://www.azadvertizer.net/azpolicyinitiativesadvertizer/1f3afdf9-d0c9-4c3d-847f-89da613e70a8.html
- 現在、ASB (Azure Security Benchmark) は MCSB (Microsoft Cloud Security Benchmark) に名称変更されているが、ポリシーファイルでは旧称が使われている。このため本資料では ASB という名称を利用する。

```bash
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_gov_change@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
TEMP_MG_TRG_ID=$(az account management-group list --query "[?displayName=='Tenant Root Group'].id" -o tsv)
 
cat > temp.json << EOF
{
    "properties": {
        "displayName": "Azure Security Benchmark",
        "scope": "${TEMP_MG_TRG_ID}",
        "notScopes": [],
        "policyDefinitionId": "/providers/Microsoft.Authorization/policySetDefinitions/1f3afdf9-d0c9-4c3d-847f-89da613e70a8",
        "enforcementMode": "Default",
        "parameters": {},
        "nonComplianceMessages": [],
        "resourceSelectors": [],
        "overrides": []
    }
}
EOF
 
az rest --method PUT --uri "${TEMP_MG_TRG_ID}/providers/Microsoft.Authorization/policyAssignments/Azure Security Benchmark?api-version=2022-06-01" --body @temp.json
 
 
 
```
