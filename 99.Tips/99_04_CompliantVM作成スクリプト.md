# Compliant VM 作成スクリプトの例

## 作業項目

- 共通
  - NIC 作成
  - VM 作成
  - ADE 適用
  - 診断設定 (プラットフォームメトリック)
  - DCE 割り当て
- 以下は Win/Linux で異なる
  - エージェント展開 : AMA, DA
  - エージェント展開 : MDE / Qualys
  - エージェント展開 : ASA, FIM
  - エージェント展開 : GC
  - UMC
  - Winのみ : ゲストアカウントリネーム、WDEG 適用、TLS 1.2 強制
  - Winのみ : GC ルール適用
  - GC CSB 適用
- Azure Backup 設定

### 作成する仮想マシン （個々の VM ごとに変更）

```bash

# VM を作成するリージョン
TEMP_LOCATION_NAME="eastus"
TEMP_LOCATION_PREFIX="eus"

TEMP_SUBSCRIPTION_ID="${SUBSCRIPTION_ID_SPOKE_A}"
TEMP_RG_NAME="rg-spokea-${TEMP_LOCATION_PREFIX}"
TEMP_VM_NAME="vm-test03-${TEMP_LOCATION_PREFIX}"
TEMP_VM_IMAGE="canonical:0001-com-ubuntu-server-focal:20_04-lts-gen2:latest"
TEMP_VM_SIZE="Standard_D2s_v3"
# canonical:0001-com-ubuntu-server-focal:20_04-lts-gen2:latest
# Win2019DataCenter
# MicrosoftWindowsServer:WindowsServer:2022-datacenter-azure-edition:latest
# MicrosoftWindowsServer:WindowsServer:2022-datacenter-azure-edition-core:latest
# canonical:0001-com-ubuntu-server-focal:20_04-lts-gen2:latest

# VM を作成するサブネット
TEMP_SUBNET_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/virtualNetworks/vnet-spokea-${TEMP_LOCATION_PREFIX}/subnets/DefaultSubnet"

ADMIN_USERNAME="azrefadmin"
ADMIN_PASSWORD="p&ssw0rdp&ssw0rd"

```

### 設定値と環境パラメータ （複数の VM で共通）

```bash

# VM 作成に必要な情報（共通設定）
TEMP_DCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionEndpoints/dce-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_DCR_AMA_WIN_NAME="dcr-law-vdc-${TEMP_LOCATION_PREFIX}-ama-win"
TEMP_DCR_AMA_LINUX_NAME="dcr-law-vdc-${TEMP_LOCATION_PREFIX}-ama-linux"
TEMP_DCR_VMI_NAME="dcr-law-vdc-${TEMP_LOCATION_PREFIX}-vmi"
TEMP_DCR_AMA_WIN_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionRules/${TEMP_DCR_AMA_WIN_NAME}"
TEMP_DCR_AMA_LINUX_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionRules/${TEMP_DCR_AMA_LINUX_NAME}"
TEMP_DCR_VMI_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionRules/${TEMP_DCR_VMI_NAME}"
TEMP_DCR_ASA_NAME="dcr-law-vdc-${TEMP_LOCATION_PREFIX}-asa" # Microsoft-Security-dcr
TEMP_DCR_ASA_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionRules/${TEMP_DCR_ASA_NAME}"
TEMP_DCR_FIM_NAME="dcr-law-vdc-${TEMP_LOCATION_PREFIX}-fim"
TEMP_DCR_FIM_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourceGroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/Microsoft.Insights/dataCollectionRules/${TEMP_DCR_FIM_NAME}"
TEMP_LAW_NAME="law-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_LAW_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/${TEMP_LAW_NAME}"
TEMP_MC_NAME="mc-daily-patching"
TEMP_MC_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourcegroups/${TEMP_RG_NAME}/providers/Microsoft.Maintenance/maintenanceConfigurations/${TEMP_MC_NAME}"

# VM 作成に必要な情報
TEMP_ADE_KV_ID="/subscriptions/${SUBSCRIPTION_ID_SPOKE_A}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.KeyVault/vaults/kv-spokea-ade-${UNIQUE_SUFFIX}-${TEMP_LOCATION_PREFIX}"

```

### NIC, VM 作成, ADE 適用

```bash

az account set -s "${TEMP_SUBSCRIPTION_ID}"

# NIC 作成
echo "Creating NIC... ${TEMP_VM_ID}"
TEMP_VM_NIC_NAME="${TEMP_VM_NAME}-nic"
TEMP_VM_NIC_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Network/networkInterfaces/${TEMP_VM_NIC_NAME}"
az network nic create --name "${TEMP_VM_NIC_NAME}" --subnet "${TEMP_SUBNET_ID}" --resource-group "${TEMP_RG_NAME}" --location "${TEMP_LOCATION_NAME}"

# VM 作成
echo "Creating VM... ${TEMP_VM_ID}"
TEMP_VM_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}"
az vm create --name "${TEMP_VM_NAME}" --image "${TEMP_VM_IMAGE}" --admin-username "${ADMIN_USERNAME}" --admin-password "${ADMIN_PASSWORD}" --nics "${TEMP_VM_NIC_NAME}" --resource-group "${TEMP_RG_NAME}" --location "${TEMP_LOCATION_NAME}" --size "${TEMP_VM_SIZE}"

# ADE 適用
# az vm encryption enable --resource-group "${TEMP_RG_NAME}" --name "${TEMP_VM_NAME}" --disk-encryption-keyvault "${TEMP_ADE_KV_ID}"
az vm encryption enable --ids "${TEMP_VM_ID}" --disk-encryption-keyvault "${TEMP_ADE_KV_ID}"

```

### 診断設定 (プラットフォームメトリック), DCE 割り当て, エージェント展開 (AMA, DA, MDE, ASA, FIM)

```bash

# 診断設定 (プラットフォームメトリック)
echo "Setting Diagnostics Settings... ${TEMP_VM_ID}"
cat <<EOF > tmp.json
{
  "properties": {
    "logs": [],
    "metrics": [
      {
        "category": "AllMetrics",
        "enabled": true,
        "retentionPolicy": {
          "days": 0,
          "enabled": false
        }
      }
    ],
    "workspaceId": "${TEMP_LAW_RESOURCE_ID}",
    "logAnalyticsDestinationType": "Dedicated"
  }
}
EOF
echo "Creating Diagnostics Settings... ${TEMP_VM_ID}"
az rest --method PUT --uri  "${TEMP_VM_ID}/providers/Microsoft.Insights/diagnosticSettings/${TEMP_LAW_NAME}?api-version=2021-05-01-preview" --body @tmp.json

# DCE 割り当て
az rest --method put --url "${TEMP_VM_ID}/providers/microsoft.insights/dataCollectionRuleAssociations/configurationAccessEndpoint?api-version=2021-04-01" --body "{ \"properties\": { \"dataCollectionEndpointId\": \"${TEMP_DCE_ID}\" } }"

# 以下は Win/Linux で異なる
# エージェント展開 : AMA, DA
echo "Installing AMA, DA, DCR-A/DCE... ${TEMP_VM_ID}"
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then # Windows OS の場合
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='DependencyAgentWindows']" -o tsv)" ]; then
    az vm identity assign --name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME}
    az vm extension set --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name "AzureMonitorWindowsAgent" --publisher "Microsoft.Azure.Monitor" --enable-auto-upgrade true
    az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --name "DependencyAgentWindows"  --publisher "Microsoft.Azure.Monitoring.DependencyAgent" --settings "{\"enableAMA\": \"true\"}"
    # MMA は不要なので削除 (基本的にインストールされていないはずだが)
    az vm extension delete --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name MicrosoftMonitoringAgent
    # DCR-A
    az monitor data-collection rule association create --name "${TEMP_DCR_AMA_WIN_NAME}-association" --rule-id ${TEMP_DCR_AMA_WIN_ID} --resource ${TEMP_VM_ID}
    az monitor data-collection rule association create --name "${TEMP_DCR_VMI_NAME}-association" --rule-id ${TEMP_DCR_VMI_ID} --resource ${TEMP_VM_ID}
  fi
fi # Windows OS
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.linuxConfiguration!=null -o tsv) == "true" ]; then # Linux OS の場合
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='DependencyAgentLinux']" -o tsv)" ]; then
    az vm identity assign --name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME}
    az vm extension set --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name "AzureMonitorLinuxAgent" --publisher "Microsoft.Azure.Monitor" --enable-auto-upgrade true
    az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --name "DependencyAgentLinux"  --publisher "Microsoft.Azure.Monitoring.DependencyAgent" --settings "{\"enableAMA\": \"true\"}"
    # OMA は不要なので削除 (基本的にインストールされていないはずだが)
    az vm extension delete --vm-name ${TEMP_VM_NAME} --resource-group ${TEMP_RG_NAME} --name OmsAgentForLinux
    # DCR-A
    az monitor data-collection rule association create --name "${TEMP_DCR_AMA_LINUX_NAME}-association" --rule-id ${TEMP_DCR_AMA_LINUX_ID} --resource ${TEMP_VM_ID}
    az monitor data-collection rule association create --name "${TEMP_DCR_VMI_NAME}-association" --rule-id ${TEMP_DCR_VMI_ID} --resource ${TEMP_VM_ID}
  fi
fi # Linux OS

# エージェント展開 : MDE / Qualys
echo "Installing MDE... ${TEMP_VM_ID}"
# MDE インストール準備
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then # Windows OS の場合
  TEMP_BASE64ENCODING_ONBOARDING_PACKAGE_WINDOWS=$(az rest --method get --url "https://management.azure.com/subscriptions/${TEMP_SUBSCRIPTION_ID}/providers/Microsoft.Security/mdeOnboardings?api-version=2021-10-01-preview" --query value[0].properties.onboardingPackageWindows -o tsv)
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='MDE.Windows']" -o tsv)" ]; then
    az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --name "MDE.Windows"  --publisher "Microsoft.Azure.AzureDefenderForServers" --enable-auto-upgrade false --no-auto-upgrade-minor-version true --settings "{\"azureResourceId\":\"${TEMP_RESOURCE_ID}\", \"defenderForServersWorkspaceId\":\"${TEMP_SUBSCRIPTION_ID}\", \"vNextEnabled\":\"true\", \"forceReOnboarding\":true, \"provisionedBy\":\"Manual\" }" --protected-settings "{ \"defenderForEndpointOnboardingScript\":\"${TEMP_BASE64ENCODING_ONBOARDING_PACKAGE_WINDOWS}\"}"
  fi
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='IaaSAntimalware']" -o tsv)" ]; then
    echo "Installing IaaSAntimalware to ${TEMP_VM_NAME}..."
cat <<EOF > tmp.json
{
    "\$schema": " https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "resources": [
    {
        "name": "${TEMP_VM_NAME}/IaaSAntimalware",
        "type": "Microsoft.Compute/virtualMachines/extensions",
        "location": "${TEMP_LOCATION_NAME}",
        "apiVersion": "2015-06-15",
        "properties": {
        "publisher": "Microsoft.Azure.Security",
        "type": "IaaSAntimalware",
        "typeHandlerVersion": "1.3",
        "autoUpgradeMinorVersion": true,
        "settings": {
            "AntimalwareEnabled": true,
            "RealtimeProtectionEnabled": true,
            "ScheduledScanSettings": {
            "isEnabled": true,
            "day": 7,
            "time": 120,
            "scanType": "Quick"
            },
            "Exclusions": {
            "Extensions": "",
            "Paths": "",
            "Processes": ""
            }
        }
        }
    }
    ]
}
EOF
    az deployment group create --name "${TEMP_VM_NAME}_IaaSAntimalware" --resource-group "${TEMP_RG_NAME}" --template-file tmp.json
  fi
fi # Windows OS
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.linuxConfiguration!=null -o tsv) == "true" ]; then # Linux OS の場合
  TEMP_BASE64ENCODING_ONBOARDING_PACKAGE_LINUX=$(az rest --method get --url "https://management.azure.com/subscriptions/${TEMP_SUBSCRIPTION_ID}/providers/Microsoft.Security/mdeOnboardings?api-version=2021-10-01-preview" --query value[0].properties.onboardingPackageLinux -o tsv)
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='MDE.Linux']" -o tsv)" ]; then
    az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --name "MDE.Linux"  --publisher "Microsoft.Azure.AzureDefenderForServers" --enable-auto-upgrade false --no-auto-upgrade-minor-version true --settings "{\"azureResourceId\":\"${TEMP_VM_ID}\", \"defenderForServersWorkspaceId\":\"${TEMP_SUBSCRIPTION_ID}\", \"vNextEnabled\":\"true\", \"forceReOnboarding\":true, \"provisionedBy\":\"Manual\" }" --protected-settings "{ \"defenderForEndpointOnboardingScript\":\"${TEMP_BASE64ENCODING_ONBOARDING_PACKAGE_LINUX}\"}"
  fi
  echo "Activating MDE realtime protection in ${TEMP_VM_NAME}..."
  az rest --method post --url "https://management.azure.com${TEMP_VM_ID}/runCommand?api-version=2018-04-01" --body "{\"commandId\":\"RunShellScript\",\"script\":[\"mdatp config real-time-protection --value enabled\\r\\n\"]}"
fi # Linux OS

# エージェント展開 : ASA
echo "Installing ASA... ${TEMP_VM_ID}"
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then # Windows OS の場合
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='AzureSecurityWindowsAgent']" -o tsv)" ]; then
    az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "AzureSecurityWindowsAgent"  --publisher "Microsoft.Azure.Security.Monitoring" --enable-auto-upgrade false --no-auto-upgrade-minor-version true
  fi
  az monitor data-collection rule association create --name "MDfS-ASA-Dcr-Association" --rule-id ${TEMP_DCR_ASA_ID} --resource ${TEMP_VM_ID}
fi # Windows OS
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.linuxConfiguration!=null -o tsv) == "true" ]; then # Linux OS の場合
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='AzureSecurityLinuxAgent']" -o tsv)" ]; then
    az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "AzureSecurityLinuxAgent"  --publisher "Microsoft.Azure.Security.Monitoring" --enable-auto-upgrade false --no-auto-upgrade-minor-version true --settings "{}"
  fi
  az monitor data-collection rule association create --name "MDfS-ASA-Dcr-Association" --rule-id ${TEMP_DCR_ASA_ID} --resource ${TEMP_VM_ID}
fi # Linux OS

# エージェント展開 : FIM
echo "Installing FIM... ${TEMP_VM_ID}"
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then # Windows OS の場合
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='ChangeTracking-Windows']" -o tsv)" ]; then
    az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "ChangeTracking-Windows" --publisher "Microsoft.Azure.ChangeTrackingAndInventory" --enable-auto-upgrade false --no-auto-upgrade-minor-version true
  fi
  az monitor data-collection rule association create --name "MDfS-FIM-Dcr-Association" --rule-id ${TEMP_DCR_FIM_ID} --resource ${TEMP_VM_ID}
fi # Windows OS
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.linuxConfiguration!=null -o tsv) == "true" ]; then # Linux OS の場合
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='ChangeTracking-Linux']" -o tsv)" ]; then
    az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "ChangeTracking-Linux"  --publisher "Microsoft.Azure.ChangeTrackingAndInventory" --enable-auto-upgrade false --no-auto-upgrade-minor-version true
  fi
  az monitor data-collection rule association create --name "MDfS-FIM-Dcr-Association" --rule-id ${TEMP_DCR_FIM_ID} --resource ${TEMP_VM_ID}
fi # Linux OS

# UMC
echo "Setting UMC... ${TEMP_VM_ID}"
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then # Windows OS の場合
  az rest --method patch --url "${TEMP_VM_ID}?api-version=2021-03-01" --body "{ \"location\": \"${TEMP_LOCATION_NAME}\", \"properties\": { \"osProfile\": { \"windowsConfiguration\": { \"patchSettings\": { \"assessmentMode\": \"AutomaticByPlatform\", \"patchMode\": \"AutomaticByPlatform\" } } } } }"
  az rest --method put --url "${TEMP_VM_ID}/providers/Microsoft.Maintenance/configurationAssignments/${TEMP_MC_NAME}Assignment?api-version=2021-09-01-preview" --body "{ \"properties\": { \"maintenanceConfigurationId\": \"${TEMP_MC_ID}\" }, \"location\": \"${TEMP_LOCATION_NAME}\" }"
fi # Windows OS
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.linuxConfiguration!=null -o tsv) == "true" ]; then # Linux OS の場合
  az rest --method patch --url "${TEMP_VM_ID}?api-version=2021-03-01" --body "{ \"location\": \"${TEMP_LOCATION_NAME}\", \"properties\": { \"osProfile\": { \"linuxConfiguration\": { \"patchSettings\": { \"assessmentMode\": \"AutomaticByPlatform\", \"patchMode\": \"AutomaticByPlatform\" } } } } }"
  az rest --method put --url "${TEMP_VM_ID}/providers/Microsoft.Maintenance/configurationAssignments/${TEMP_MC_NAME}Assignment?api-version=2021-09-01-preview" --body "{ \"properties\": { \"maintenanceConfigurationId\": \"${TEMP_MC_ID}\" }, \"location\": \"${TEMP_LOCATION_NAME}\" }"
fi # Linux OS

# エージェント展開 : GC
echo "Installing GC... ${TEMP_VM_ID}"
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then # Windows OS の場合
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='AzurePolicyforWindows']" -o tsv)" ]; then
  az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "ConfigurationforWindows" --publisher "Microsoft.GuestConfiguration" --extension-instance-name "AzurePolicyforWindows" --enable-auto-upgrade false --no-auto-upgrade-minor-version true
  fi
fi # Windows OS
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.linuxConfiguration!=null -o tsv) == "true" ]; then # Linux OS の場合
  if [ -z "$(az vm extension list --vm-name "${TEMP_VM_NAME}" --resource-group ${TEMP_RG_NAME} --query "[?name=='AzurePolicyforLinux']" -o tsv)" ]; then
  az vm extension set --vm-name "${TEMP_VM_NAME}" --resource-group "${TEMP_RG_NAME}" --name "ConfigurationforLinux" --publisher "Microsoft.GuestConfiguration" --extension-instance-name "AzurePolicyforLinux" --enable-auto-upgrade false --no-auto-upgrade-minor-version true
  fi
fi # Linux OS

```

### ゲスト構成・OS ハードニング

```bash

# Winのみ : ゲストアカウントリネーム、WDEG 適用、TLS 1.2 強制
echo "Changing OS configuration... ${TEMP_VM_ID}"
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then # Windows OS の場合
  az rest --method post --url "${TEMP_VM_ID}/runCommand?api-version=2018-04-01" --body "{\"commandId\":\"RunPowerShellScript\",\"script\":[\"wmic useraccount where \\\"name='Guest'\\\" rename 'LocalGuest'\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids be9ba2d9-53ea-4cdc-84e5-9b1eeee46550 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids b2b3f03d-6a65-4f7b-a9c7-1c7ef74a9ba4 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 9e6c4e1f-7d60-472f-ba1a-a39ef669e4b2 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids d4f940ab-401b-4efc-aadc-ad5f3c50688a -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids d3e037e1-3eb8-44c8-a917-57927947596d -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 5beb7efe-fd9a-4556-801d-275e5ffc04cc -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 3b576869-a4ec-4529-8536-b80a7769e899 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 26190899-1602-49e8-8b27-eb1d0a1ce869 -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 92e97fa1-2edf-4476-bdd6-9dd0b4dddc7b -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 7674ba52-37eb-4a4f-a9a1-f0f9a1619a2c -AttackSurfaceReductionRules_Actions Enabled\\r\\nAdd-MpPreference -AttackSurfaceReductionRules_Ids 75668c1f-73b5-4cf0-bb93-3ecf5cb7cc84 -AttackSurfaceReductionRules_Actions Enabled\\r\\nSet-MpPreference -EnableControlledFolderAccess Enabled\\r\\necho done.\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 2.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 2.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 3.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\SSL 3.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\PCT 1.0\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\PCT 1.0\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\Multi-Protocol Unified Hello\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\Multi-Protocol Unified Hello\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.1\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.1\\Server' -Name 'Enabled' -Value '0' -PropertyType 'DWord' -Force | Out-Null\\r\\nNew-Item 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.2\\Server' -Force | Out-Null\\r\\nNew-ItemProperty -Path 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL\\Protocols\\TLS 1.2\\Server' -Name 'Enabled' -Value '1' -PropertyType 'DWord' -Force | Out-Null\\r\\necho done.\"]}"
fi # Windows OS

# Winのみ : GC ルール適用
echo "Applying GC rules for Windows... ${TEMP_VM_ID}"
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then # Windows OS の場合

# AuditSecureProtocol
TEMP_GC_ASSIGNMENT_NAME="AuditSecureProtocol"
TEMP_GC_ASSIGNMENT_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.GuestConfiguration/guestConfigurationAssignments/${TEMP_GC_ASSIGNMENT_NAME}"
cat <<EOF > tmp.json
{
  "location": "${TEMP_LOCATION_NAME}",
  "name": "${TEMP_GC_ASSIGNMENT_NAME}",
  "properties": {
    "guestConfiguration": {
      "name": "${TEMP_GC_ASSIGNMENT_NAME}",
      "version": "1.*",
      "assignmentType": null,
      "configurationParameter": [
        {
          "name": "[SecureWebServer]s1;MinimumTLSVersion",
          "value": "1.2"
        }
      ],
      "configurationSetting": {
        "actionAfterReboot": "",
        "allowModuleOverwrite": false,
        "configurationMode": "MonitorOnly",
        "configurationModeFrequencyMins": 15,
        "rebootIfNeeded": true,
        "refreshFrequencyMins": 5
      }
    }
  }
}
EOF
echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}" # 赤字で表示されるのを避けるために、いったん結果を変数に格納してから表示
TEMP_RESPONSE=$(az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method DELETE)
echo $TEMP_RESPONSE
echo "Creating GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method PUT --body @tmp.json

# AzureWindowsVMEncryptionCompliance
TEMP_GC_ASSIGNMENT_NAME="AzureWindowsVMEncryptionCompliance"
TEMP_GC_ASSIGNMENT_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.GuestConfiguration/guestConfigurationAssignments/${TEMP_GC_ASSIGNMENT_NAME}"
cat <<EOF > tmp.json
{
  "location": "${TEMP_LOCATION_NAME}",
  "name": "${TEMP_GC_ASSIGNMENT_NAME}",
  "properties": {
    "guestConfiguration": {
      "name": "${TEMP_GC_ASSIGNMENT_NAME}",
      "version": "1.*",
      "assignmentType": null,
      "configurationParameter": [],
      "configurationSetting": {
        "actionAfterReboot": "",
        "allowModuleOverwrite": false,
        "configurationMode": "MonitorOnly",
        "configurationModeFrequencyMins": 15,
        "rebootIfNeeded": true,
        "refreshFrequencyMins": 5
      }
    }
  }
}
EOF
echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}" # 赤字で表示されるのを避けるために、いったん結果を変数に格納してから表示
TEMP_RESPONSE=$(az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method DELETE)
echo $TEMP_RESPONSE
echo "Creating GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method PUT --body @tmp.json

# WindowsDefenderExploitGuard
TEMP_GC_ASSIGNMENT_NAME="WindowsDefenderExploitGuard"
TEMP_GC_ASSIGNMENT_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.GuestConfiguration/guestConfigurationAssignments/${TEMP_GC_ASSIGNMENT_NAME}"
cat <<EOF > tmp.json
{
  "location": "${TEMP_LOCATION_NAME}",
  "name": "${TEMP_GC_ASSIGNMENT_NAME}",
  "properties": {
    "guestConfiguration": {
      "name": "${TEMP_GC_ASSIGNMENT_NAME}",
      "version": "1.*",
      "assignmentType": null,
      "configurationParameter": [
        {
          "name": "[WindowsDefenderExploitGuard]WindowsDefenderExploitGuard1;NotAvailableMachineState",
          "value": "Compliant"
        }
      ],
      "configurationSetting": {
        "actionAfterReboot": "",
        "allowModuleOverwrite": false,
        "configurationMode": "MonitorOnly",
        "configurationModeFrequencyMins": 15,
        "rebootIfNeeded": true,
        "refreshFrequencyMins": 5
      }
    }
  }
}
EOF
echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}" # 赤字で表示されるのを避けるために、いったん結果を変数に格納してから表示
TEMP_RESPONSE=$(az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method DELETE)
echo $TEMP_RESPONSE
echo "Creating GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method PUT --body @tmp.json

fi # Windows OS

# GC CSB 適用
echo "Applying CSB... ${TEMP_VM_ID}"
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.windowsConfiguration!=null -o tsv) == "true" ]; then # Windows OS の場合
# AzureWindowsBaseline
TEMP_GC_ASSIGNMENT_NAME="AzureWindowsBaseline"
TEMP_GC_ASSIGNMENT_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.GuestConfiguration/guestConfigurationAssignments/${TEMP_GC_ASSIGNMENT_NAME}"
cat <<EOF > tmp.json
{
  "location": "${TEMP_LOCATION_NAME}",
  "name": "${TEMP_GC_ASSIGNMENT_NAME}",
  "properties": {
    "guestConfiguration": {
      "name": "${TEMP_GC_ASSIGNMENT_NAME}",
      "version": "1.*",
      "assignmentType": "ApplyAndMonitor",
      "configurationParameter": [
      ],
      "configurationSetting": {
        "actionAfterReboot": "",
        "allowModuleOverwrite": false,
        "configurationMode": "ApplyAndMonitor",
        "configurationModeFrequencyMins": 15,
        "rebootIfNeeded": true,
        "refreshFrequencyMins": 5
      }
    }
  }
}
EOF
echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}" # 赤字で表示されるのを避けるために、いったん結果を変数に格納してから表示
TEMP_RESPONSE=$(az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method DELETE)
echo $TEMP_RESPONSE
echo "Creating GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method PUT --body @tmp.json
fi # Windows OS
if [ $(az vm show --ids "${TEMP_VM_ID}" --query osProfile.linuxConfiguration!=null -o tsv) == "true" ]; then # Linux OS の場合
# AzureLinuxBaseline
TEMP_GC_ASSIGNMENT_NAME="AzureLinuxBaseline"
TEMP_GC_ASSIGNMENT_ID="/subscriptions/${TEMP_SUBSCRIPTION_ID}/resourceGroups/${TEMP_RG_NAME}/providers/Microsoft.Compute/virtualMachines/${TEMP_VM_NAME}/providers/Microsoft.GuestConfiguration/guestConfigurationAssignments/${TEMP_GC_ASSIGNMENT_NAME}"
cat <<EOF > tmp.json
{
  "location": "${TEMP_LOCATION_NAME}",
  "name": "${TEMP_GC_ASSIGNMENT_NAME}",
  "properties": {
    "guestConfiguration": {
      "name": "${TEMP_GC_ASSIGNMENT_NAME}",
      "version": "1.*",
      "assignmentType": "ApplyAndMonitor",
      "configurationParameter": [
      ],
      "configurationSetting": {
        "actionAfterReboot": "",
        "allowModuleOverwrite": false,
        "configurationMode": "ApplyAndMonitor",
        "configurationModeFrequencyMins": 15,
        "rebootIfNeeded": true,
        "refreshFrequencyMins": 5
      }
    }
  }
}
EOF
echo "Deleting existing GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}" # 赤字で表示されるのを避けるために、いったん結果を変数に格納してから表示
TEMP_RESPONSE=$(az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method DELETE)
echo $TEMP_RESPONSE
echo "Creating GC ${TEMP_GC_ASSIGNMENT_NAME} rule on ${TEMP_VM_NAME}"
az rest --uri "${TEMP_GC_ASSIGNMENT_ID}?api-version=2022-01-25" --method PUT --body @tmp.json
fi # Linux OS

# VM 再起動
echo "Restarting VM... ${TEMP_VM_NAME}"
az vm restart --force --ids ${TEMP_VM_ID} --no-wait

```

### Azure Backup 設定

※ テスト目的のため割愛
