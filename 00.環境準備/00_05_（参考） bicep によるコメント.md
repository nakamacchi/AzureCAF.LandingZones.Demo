# （参考） bicep によるコメント

```bash
 
# （参考） カスタムロールの保守を行う上で、各権限が何の目的のものなのかを記録しておく必要がある
# ARM テンプレートの JSON だとコメントが書けないため、bicep で維持する方法がオススメ
 
cat > temp.bicep << EOF
targetScope = 'subscription'
resource CR_NAMEUUID_SPOKE_DEV_ON_SPOKE_DIFF 'Microsoft.Authorization/roleDefinitions@2018-07-01' = {
  name: '${CR_NAMEUUID_SPOKE_DEV_ON_SPOKE_DIFF}'
  properties: {
    roleName: '${CR_ROLENAME_SPOKE_DEV_ON_SPOKE_DIFF}'
    description: '${CR_ROLENAME_SPOKE_DEV_ON_SPOKE_DIFF}'
    type: 'customRole'
    permissions: [
      {
        actions: []
        notActions: [
          // Network 関連の権限を削除
          'Microsoft.Authorization/*/write'
          'Microsoft.Authorization/*/delete'
          'Microsoft.Authorization/elevateAccess/Action'
          'Microsoft.Blueprint/blueprintAssignments/write'
          'Microsoft.Blueprint/blueprintAssignments/delete'
          'Microsoft.Compute/galleries/share/action'
          'Microsoft.Network/*/write'
          'Microsoft.Network/*/delete'
          'Microsoft.ClassicNetwork/*'
        ]
        dataActions: []
        notDataActions: []
      }
    ]
    assignableScopes: [
      '${TEMP_MG_TRG_ID}'
    ]
  }
}
 
resource CR_NAMEUUID_SPOKE_OPS_ON_SPOKE 'Microsoft.Authorization/roleDefinitions@2018-07-01' = {
  name: '${CR_NAMEUUID_SPOKE_OPS_ON_SPOKE}'
  properties: {
    roleName: 'Custom Role for Spoke Operator on Spoke'
    description: 'Custom Role for Spoke Operator on Spoke'
    type: 'customRole'
    permissions: [
      {
        actions: [
          '*/read'
          // 仮想マシン関連の日常的な作業のための権限
          'Microsoft.Compute/virtualMachines/start/action'                        // VM の起動
          'Microsoft.Compute/virtualMachines/powerOff/action'                        // VM の停止
          'Microsoft.Compute/virtualMachines/deallocate/action'                // VM の切断
          'Microsoft.Compute/virtualMachines/performMaintenance/action'        // VM のメンテナンス
          'Microsoft.Compute/virtualMachines/restart/action'                        // VM の再起動
          'Microsoft.Compute/virtualMachines/installPatches/action'                // VM へのパッチ適用
          'Microsoft.Compute/virtualMachines/assessPatches/action'                // VM のパッチ確認
          'Microsoft.Compute/virtualMachines/cancelPatchInstallation/action' // VM のパッチインストールのキャンセル
          // SQL VM 関連の日常的な作業のための権限
          'Microsoft.SqlVirtualMachine/sqlVirtualMachines/startAssessment/action'
          'Microsoft.SqlVirtualMachine/sqlVirtualMachines/troubleshoot/action'
        ]
        notActions: []
        dataActions: []
        notDataActions: []
      }
    ]
    assignableScopes: [
      '${TEMP_MG_TRG_ID}'
    ]
  }
}
 
resource CR_NAMEUUID_SPOKE_OPS_ON_MGMT 'Microsoft.Authorization/roleDefinitions@2018-07-01' = {
  name: '${CR_NAMEUUID_SPOKE_OPS_ON_MGMT}'
  properties: {
    roleName: '${CR_ROLENAME_SPOKE_OPS_ON_MGMT}'
    description: '${CR_ROLENAME_SPOKE_OPS_ON_MGMT}'
    type: 'customRole'
    permissions: [
      {
        actions: [
          'microsoft.operationalinsights/workspaces/features/generateMap/read'        // VM Insights の利用に必要
        ]
        notActions: []
        dataActions: []
        notDataActions: []
      }
    ]
    assignableScopes: [
      '${TEMP_MG_TRG_ID}'
    ]
  }
}
 
resource CR_NAMEUUID_NW_CNG_ON_TRG_DIFF 'Microsoft.Authorization/roleDefinitions@2018-07-01' = {
  name: '${CR_NAMEUUID_NW_CNG_ON_TRG_DIFF}'
  properties: {
    roleName: '${CR_ROLENAME_NW_CNG_ON_TRG_DIFF}'
    description: '${CR_ROLENAME_NW_CNG_ON_TRG_DIFF}'
    type: 'customRole'
    permissions: [
      {
        actions: [
          '*/PrivateEndpointConnectionsApproval/action'                // Private Endpoint の作成に必要
          'Microsoft.Web/sites/write'                                                // VNET 統合に必要
          'Microsoft.Web/sites/config/list/action'                        // VNET 統合時の設定書き込みに必要
          'Microsoft.Web/sites/config/write'                                // VNET 統合時の設定書き込みに必要
          'Microsoft.Web/sites/restart/action'                                // VNET 統合時の設定反映に必要
        ]
        notActions: []
        dataActions: []
        notDataActions: []
      }
    ]
    assignableScopes: [
      '${TEMP_MG_TRG_ID}'
    ]
  }
}
 
resource CR_NAMEUUID_MGMT_OPS_ON_TRG 'Microsoft.Authorization/roleDefinitions@2018-07-01' = {
  name: '${CR_NAMEUUID_MGMT_OPS_ON_TRG}'
  properties: {
    roleName: '${CR_ROLENAME_MGMT_OPS_ON_TRG}'
    description: '${CR_ROLENAME_MGMT_OPS_ON_TRG}'
    type: 'customRole'
    permissions: [
      {
        actions: [
          'Microsoft.Compute/virtualMachines/assessPatches/action'
        ]
        notActions: []
        dataActions: []
        notDataActions: []
      }
    ]
    assignableScopes: [
      '${TEMP_MG_TRG_ID}'
    ]
  }
}
 
resource CR_NAMEUUID_GOV_CNG_ON_TRG_DIFF 'Microsoft.Authorization/roleDefinitions@2018-07-01' = {
  name: '${CR_NAMEUUID_GOV_CNG_ON_TRG_DIFF}'
  properties: {
    roleName: '${CR_ROLENAME_GOV_CNG_ON_TRG_DIFF}dmin, Monitoring Contributor'
    description: '${CR_ROLENAME_GOV_CNG_ON_TRG_DIFF}dmin, Monitoring Contributor'
    type: 'customRole'
    permissions: [
      {
        actions: [
          'Microsoft.Compute/virtualMachines/write'                                                        // VM への介入に必要
          'Microsoft.Compute/virtualMachines/extensions/write'                                        // VM 拡張の修正に必要
          'Microsoft.Compute/virtualMachines/extensions/delete'                                // VM 拡張の修正に必要
          'Microsoft.Compute/virtualMachines/runCommand/action'                                // VM の EDR 有効化に必要
          'Microsoft.Compute/virtualMachines/installPatches/action'                                // VM への強制パッチ適用に必要
          'Microsoft.Compute/virtualMachines/assessPatches/action'                                // VM のパッチ適用状況確認に必要
          'Microsoft.Compute/virtualMachines/cancelPatchInstallation/action'                // VM へのパッチ適用のキャンセルに必要
          'Microsoft.Compute/virtualMachines/assessPatches/action'                                // VM のパッチ適用確認に必要
          'Microsoft.Maintenance/maintenanceConfigurations/write'                                // VM のメンテナンス構成の設定に必要
          'Microsoft.Maintenance/configurationAssignments/write'                                // VM のメンテナンス構成の設定に必要
          'Microsoft.SqlVirtualMachine/sqlVirtualMachines/write'                                // SQL VM の確認に必要
          '*/register/action'                                                                                        // 上記機能の有効化に必要
        ]
        notActions: []
        dataActions: []
        notDataActions: []
      }
    ]
    assignableScopes: [
      '${TEMP_MG_TRG_ID}'
    ]
  }
}
EOF
 
az bicep build --file temp.bicep
TEMP=(${TEMP_MG_TRG_ID//\// })
az deployment mg create --location ${LOCATION_NAMES[0]} --name "CustomRoleCreate" --template-file temp.json --management-group-id "${TEMP[3]}"
 
 
```
