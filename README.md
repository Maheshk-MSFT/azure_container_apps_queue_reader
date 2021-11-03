# azure_container_apps_queue_reader
azure_container_apps_queue_reader

RESOURCE_GROUP="mikky-containerapps"
LOCATION="canadacentral"
CONTAINERAPPS_ENVIRONMENT="containerapps-env"
LOG_ANALYTICS_WORKSPACE="containerapps-logs"
STORAGE_ACCOUNT="containerappsstorage202"

az extension add --source https://workerappscliextension.blob.core.windows.net/azure-cli-extension/containerapp-0.2.0-py2.py3-none-any.whl

az provider register --namespace Microsoft.Web

az group create --name $RESOURCE_GROUP --location "$LOCATION"

az monitor log-analytics workspace create --resource-group $RESOURCE_GROUP --workspace-name $LOG_ANALYTICS_WORKSPACE

LOG_ANALYTICS_WORKSPACE_CLIENT_ID=`az monitor log-analytics workspace show --query customerId -g $RESOURCE_GROUP -n $LOG_ANALYTICS_WORKSPACE --out json | tr -d '"'`

LOG_ANALYTICS_WORKSPACE_CLIENT_SECRET=`az monitor log-analytics workspace get-shared-keys --query primarySharedKey -g $RESOURCE_GROUP -n $LOG_ANALYTICS_WORKSPACE --out json | tr -d '"'`

az containerapp env create --name $CONTAINERAPPS_ENVIRONMENT --resource-group $RESOURCE_GROUP --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_CLIENT_ID --logs-workspace-key $LOG_ANALYTICS_WORKSPACE_CLIENT_SECRET --location "$LOCATION"

az storage account create --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --location "$LOCATION" --sku Standard_RAGRS --kind StorageV2

QUEUE_CONNECTION_STRING=`az storage account show-connection-string -g $RESOURCE_GROUP --name $STORAGE_ACCOUNT --query connectionString --out json | tr -d '"'`

az storage queue create --name "myqueue" --account-name $STORAGE_ACCOUNT --connection-string $QUEUE_CONNECTION_STRING

az storage message put --content "Hello Queue Reader App" --queue-name "myqueue" --connection-string $QUEUE_CONNECTION_STRING

az deployment group create --resource-group "$RESOURCE_GROUP" --template-file ./queue.json --parameters environment_name="$CONTAINERAPPS_ENVIRONMENT" queueconnection="$QUEUE_CONNECTION_STRING" location="$LOCATION"

az monitor log-analytics query --workspace $LOG_ANALYTICS_WORKSPACE_CLIENT_ID --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == 'queuereader' and Log_s contains 'Message ID'" --out table

