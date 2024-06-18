# Defender for SQL 用の事前準備

```bash

# 共通基盤管理チーム／① 初期構築時の作業アカウントに切り替え
if ${FLAG_USE_SOD} ; then az account clear ; az login -u "user_plat_dev@${PRIMARY_DOMAIN_NAME}" -p "${ADMIN_PASSWORD}" ; fi
 
# 運用管理サブスクリプションに切り替え
az account set -s "${SUBSCRIPTION_ID_MGMT}"
 
# LAWS へのソリューションのインストール
 
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_LAW_NAME="law-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_LAW_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/${TEMP_LAW_NAME}"
TEMP_DCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.insights/datacollectionendpoints/dce-vdc-${TEMP_LOCATION_PREFIX}"
 
cat <<EOF > tmp.json
{
  "\$schema": " https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "solutions": {
            "type": "array",
            "defaultValue": [
                "SQLAdvancedThreatProtection",
                "SQLVulnerabilityAssessment",
            ]
        },
        "workspaceName": {
            "type": "string",
            "defaultValue": "${TEMP_LAW_NAME}"
        },
        "workspaceId": {
            "type": "string",
            "defaultValue": "${TEMP_LAW_RESOURCE_ID}"
        }
    },
    "resources": [
        {
            "name": "[concat(parameters('solutions')[CopyIndex()], '(', parameters('workspaceName'), ')')]",
            "type": "Microsoft.OperationsManagement/solutions",
            "apiVersion": "2015-11-01-preview",
            "location": "${TEMP_LOCATION_NAME}",
            "tags": {},
            "plan": {
                "name": "[concat(parameters('solutions')[CopyIndex()], '(', parameters('workspaceName'), ')')]",
                "publisher": "Microsoft",
                "promotionCode": "",
                "product": "[concat('OMSGallery/', parameters('solutions')[CopyIndex()])]"
            },
            "properties": {
                "workspaceResourceId": "${TEMP_LAW_RESOURCE_ID}",
                "containedResources": [
                    "[concat(parameters('workspaceId'), '/views/', parameters('solutions')[CopyIndex()], '(', parameters('workspaceName'), ')')]"
                ]
            },
            "copy": {
                "name": "solutionAssignment",
                "count": "[length(parameters('solutions'))]"
            }
        }
    ]
}
EOF
az deployment group create --name "SolutionDeploy-${TEMP_LAW_NAME}" --resource-group $TEMP_RG_NAME --template-file tmp.json
 
done # LOCATION

# DCR を MGMT サブスクリプション側に作成しておく
for i in ${VDC_NUMBERS}; do
TEMP_LOCATION_NAME=${LOCATION_NAMES[$i]}
TEMP_LOCATION_PREFIX=${LOCATION_PREFIXS[$i]}

TEMP_RG_NAME="rg-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_LAW_NAME="law-vdc-${TEMP_LOCATION_PREFIX}"
TEMP_LAW_RESOURCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.operationalinsights/workspaces/${TEMP_LAW_NAME}"
TEMP_DCE_ID="/subscriptions/${SUBSCRIPTION_ID_MGMT}/resourcegroups/rg-vdc-${TEMP_LOCATION_PREFIX}/providers/microsoft.insights/datacollectionendpoints/dce-vdc-${TEMP_LOCATION_PREFIX}"

TEMP_DCR_SQL_WIN_NAME="dcr-law-vdc-${TEMP_LOCATION_PREFIX}-sql-win"
 
cat <<EOF > dcr.json
{
    "location": "${TEMP_LOCATION_NAME}",
    "properties": {
        "description": "Data collection rule for SQL",
        "dataCollectionEndpointId": "${TEMP_DCE_ID}",
        "dataSources": {
            "extensions": [
              {
                "extensionName": "MicrosoftDefenderForSQL",
                "name": "MicrosoftDefenderForSQL",
                "streams": [
                          "Microsoft-DefenderForSqlAlerts",
                          "Microsoft-DefenderForSqlLogins",
                          "Microsoft-DefenderForSqlTelemetry",
                          "Microsoft-DefenderForSqlScanEvents",
                          "Microsoft-DefenderForSqlScanResults",
                          "Microsoft-SqlAtpStatus-DefenderForSql"
                        ],
                "extensionSettings": {
                  "enableCollectionOfSqlQueriesForSecurityResearch": false
                }
              }
            ]
        },
        "destinations": {
            "logAnalytics": [
                {
                    "workspaceResourceId": "${TEMP_LAW_RESOURCE_ID}",
                    "name": "LogAnalyticsDest"
                }
            ]
        },
        "dataFlows": [
          {
            "streams": [
              "Microsoft-DefenderForSqlAlerts",
              "Microsoft-DefenderForSqlLogins",
              "Microsoft-DefenderForSqlTelemetry",
              "Microsoft-DefenderForSqlScanEvents",
              "Microsoft-DefenderForSqlScanResults",
              "Microsoft-SqlAtpStatus-DefenderForSql"
            ],
            "destinations": [
                "LogAnalyticsDest"
              ]
          }
        ]
    }
}
EOF
az monitor data-collection rule create --name ${TEMP_DCR_SQL_WIN_NAME} --resource-group "${TEMP_RG_NAME}" --location "${TEMP_LOCATION_NAME}" --rule-file dcr.json
 
done # TEMP_LOCATION

```
