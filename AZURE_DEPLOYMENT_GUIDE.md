# Azure Deployment Guide

## Overview

This guide provides step-by-step instructions for deploying the modernized Java applications to Azure. It covers infrastructure provisioning, application deployment, and post-deployment validation.

## Prerequisites

### Required Tools
- **Azure CLI** (version 2.50.0 or later)
  ```bash
  az --version
  az login
  ```
- **Docker** (version 20.10 or later)
  ```bash
  docker --version
  ```
- **Maven** (version 3.8.0 or later)
  ```bash
  mvn --version
  ```
- **Java JDK** (versions 17 and 21)
  ```bash
  java --version
  ```
- **Git**
  ```bash
  git --version
  ```

### Azure Subscription Setup
1. Create or use existing Azure subscription
2. Ensure you have appropriate permissions:
   - Contributor or Owner role on subscription
   - Ability to create service principals
   - Access to create Azure AD app registrations

3. Set default subscription:
   ```bash
   az account set --subscription "YOUR_SUBSCRIPTION_ID"
   ```

## Deployment Architecture

### Resource Naming Convention
```
Format: {app-name}-{environment}-{region}-{resource-type}

Examples:
- assetmanager-prod-eastus-app (Container App)
- assetmanager-prod-eastus-psql (PostgreSQL Server)
- assetmanager-prod-eastus-sb (Service Bus)
- assetmanager-prod-eastus-st (Storage Account)
```

### Environment Strategy
- **Development**: dev-eastus
- **Staging**: staging-eastus
- **Production**: prod-eastus (with DR in westus)

## Infrastructure Provisioning

### Option 1: Using Azure CLI and Bicep

#### Step 1: Create Resource Group
```bash
# Set variables
RESOURCE_GROUP="java-migration-prod-rg"
LOCATION="eastus"
ENVIRONMENT="prod"

# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --tags environment=$ENVIRONMENT project=java-migration
```

#### Step 2: Create Azure Container Registry
```bash
ACR_NAME="javamigrationprodacr"

az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Standard \
  --admin-enabled false \
  --location $LOCATION

# Enable managed identity
az acr update \
  --name $ACR_NAME \
  --resource-group $RESOURCE_GROUP \
  --anonymous-pull-enabled false
```

#### Step 3: Create Azure Database for PostgreSQL
```bash
PSQL_SERVER_NAME="javamigration-prod-psql"
PSQL_ADMIN_USER="javamigadmin"
PSQL_DATABASE_NAME="assetmanager"

# Create PostgreSQL Flexible Server
az postgres flexible-server create \
  --resource-group $RESOURCE_GROUP \
  --name $PSQL_SERVER_NAME \
  --location $LOCATION \
  --admin-user $PSQL_ADMIN_USER \
  --admin-password "$(openssl rand -base64 32)" \
  --sku-name Standard_D2s_v3 \
  --tier GeneralPurpose \
  --storage-size 128 \
  --version 15 \
  --high-availability Disabled \
  --backup-retention 7 \
  --public-access 0.0.0.0-255.255.255.255

# Create database
az postgres flexible-server db create \
  --resource-group $RESOURCE_GROUP \
  --server-name $PSQL_SERVER_NAME \
  --database-name $PSQL_DATABASE_NAME

# Configure firewall (for development - adjust for production)
az postgres flexible-server firewall-rule create \
  --resource-group $RESOURCE_GROUP \
  --name $PSQL_SERVER_NAME \
  --rule-name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Enable Azure AD authentication
az postgres flexible-server ad-admin create \
  --resource-group $RESOURCE_GROUP \
  --server-name $PSQL_SERVER_NAME \
  --display-name "PostgreSQL Admin" \
  --object-id "YOUR_AZURE_AD_OBJECT_ID"
```

#### Step 4: Create Azure Storage Account
```bash
STORAGE_ACCOUNT_NAME="javamigprodstorage"

az storage account create \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot \
  --https-only true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

# Create containers
az storage container create \
  --name images \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login

az storage container create \
  --name thumbnails \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login
```

#### Step 5: Create Azure Service Bus
```bash
SERVICEBUS_NAMESPACE="javamigration-prod-sb"
QUEUE_NAME="image-processing"

az servicebus namespace create \
  --resource-group $RESOURCE_GROUP \
  --name $SERVICEBUS_NAMESPACE \
  --location $LOCATION \
  --sku Standard

# Create queue
az servicebus queue create \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $SERVICEBUS_NAMESPACE \
  --name $QUEUE_NAME \
  --max-size 1024 \
  --default-message-time-to-live P14D \
  --enable-dead-lettering-on-message-expiration true

# Create retry queue with delayed delivery
az servicebus queue create \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $SERVICEBUS_NAMESPACE \
  --name "${QUEUE_NAME}-retry" \
  --max-size 1024 \
  --default-message-time-to-live P14D \
  --enable-dead-lettering-on-message-expiration true
```

#### Step 6: Create Azure Container Apps Environment
```bash
CONTAINERAPPS_ENVIRONMENT="javamigration-prod-env"
LOG_ANALYTICS_WORKSPACE="javamigration-prod-logs"

# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_ANALYTICS_WORKSPACE \
  --location $LOCATION

# Get workspace ID and key
LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_ANALYTICS_WORKSPACE \
  --query customerId -o tsv)

LOG_ANALYTICS_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_ANALYTICS_WORKSPACE \
  --query primarySharedKey -o tsv)

# Create Container Apps environment
az containerapp env create \
  --name $CONTAINERAPPS_ENVIRONMENT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_ID \
  --logs-workspace-key $LOG_ANALYTICS_KEY
```

#### Step 7: Create Application Insights
```bash
APP_INSIGHTS_NAME="javamigration-prod-insights"

az monitor app-insights component create \
  --app $APP_INSIGHTS_NAME \
  --location $LOCATION \
  --resource-group $RESOURCE_GROUP \
  --workspace $LOG_ANALYTICS_WORKSPACE

# Get instrumentation key
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app $APP_INSIGHTS_NAME \
  --resource-group $RESOURCE_GROUP \
  --query instrumentationKey -o tsv)
```

### Option 2: Using Bicep Templates

#### Step 1: Create main.bicep
```bicep
// Save as infrastructure/main.bicep
param location string = resourceGroup().location
param environment string = 'prod'
param projectName string = 'javamigration'

// Generate unique names
var uniqueSuffix = substring(uniqueString(resourceGroup().id), 0, 6)
var acrName = '${projectName}${environment}acr${uniqueSuffix}'
var psqlServerName = '${projectName}-${environment}-psql-${uniqueSuffix}'
var storageAccountName = '${projectName}${environment}st${uniqueSuffix}'
var serviceBusNamespace = '${projectName}-${environment}-sb-${uniqueSuffix}'
var containerAppsEnvironment = '${projectName}-${environment}-env'
var logAnalyticsWorkspace = '${projectName}-${environment}-logs'
var appInsightsName = '${projectName}-${environment}-insights'

module acr 'modules/acr.bicep' = {
  name: 'acrDeployment'
  params: {
    name: acrName
    location: location
  }
}

module postgresql 'modules/postgresql.bicep' = {
  name: 'postgresqlDeployment'
  params: {
    serverName: psqlServerName
    location: location
    administratorLogin: 'javamigadmin'
  }
}

module storage 'modules/storage.bicep' = {
  name: 'storageDeployment'
  params: {
    storageAccountName: storageAccountName
    location: location
  }
}

module servicebus 'modules/servicebus.bicep' = {
  name: 'servicebusDeployment'
  params: {
    namespaceName: serviceBusNamespace
    location: location
  }
}

module containerApps 'modules/containerApps.bicep' = {
  name: 'containerAppsDeployment'
  params: {
    environmentName: containerAppsEnvironment
    logAnalyticsName: logAnalyticsWorkspace
    appInsightsName: appInsightsName
    location: location
  }
}

output acrLoginServer string = acr.outputs.loginServer
output postgresqlFqdn string = postgresql.outputs.fqdn
output storageAccountName string = storage.outputs.storageAccountName
output serviceBusEndpoint string = servicebus.outputs.serviceBusEndpoint
```

#### Step 2: Deploy Bicep Template
```bash
az deployment group create \
  --resource-group $RESOURCE_GROUP \
  --template-file infrastructure/main.bicep \
  --parameters environment=prod projectName=javamigration
```

## Application Deployment

### Asset Manager Application

#### Step 1: Build and Push Web Container
```bash
# Navigate to web module
cd asset-manager/web

# Build application
mvn clean package -DskipTests

# Build Docker image
docker build -t assetmanager-web:latest .

# Tag for ACR
docker tag assetmanager-web:latest $ACR_NAME.azurecr.io/assetmanager-web:latest

# Login to ACR
az acr login --name $ACR_NAME

# Push image
docker push $ACR_NAME.azurecr.io/assetmanager-web:latest
```

#### Step 2: Build and Push Worker Container
```bash
# Navigate to worker module
cd ../worker

# Build application
mvn clean package -DskipTests

# Build Docker image
docker build -t assetmanager-worker:latest .

# Tag for ACR
docker tag assetmanager-worker:latest $ACR_NAME.azurecr.io/assetmanager-worker:latest

# Push image
docker push $ACR_NAME.azurecr.io/assetmanager-worker:latest
```

#### Step 3: Create Managed Identity
```bash
IDENTITY_NAME="assetmanager-identity"

# Create user-assigned managed identity
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME

# Get identity details
IDENTITY_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query id -o tsv)

IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query clientId -o tsv)

IDENTITY_PRINCIPAL_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query principalId -o tsv)
```

#### Step 4: Assign RBAC Roles
```bash
# Get resource IDs
STORAGE_ACCOUNT_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

SERVICEBUS_ID=$(az servicebus namespace show \
  --name $SERVICEBUS_NAMESPACE \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Assign Storage Blob Data Contributor
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ACCOUNT_ID

# Assign Service Bus Data Owner
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "Azure Service Bus Data Owner" \
  --scope $SERVICEBUS_ID

# Assign ACR Pull
ACR_ID=$(az acr show \
  --name $ACR_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "AcrPull" \
  --scope $ACR_ID
```

#### Step 5: Deploy Web Container App
```bash
# Get PostgreSQL connection string
PSQL_FQDN=$(az postgres flexible-server show \
  --resource-group $RESOURCE_GROUP \
  --name $PSQL_SERVER_NAME \
  --query fullyQualifiedDomainName -o tsv)

# Deploy web app
az containerapp create \
  --name assetmanager-web \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --image $ACR_NAME.azurecr.io/assetmanager-web:latest \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 10 \
  --cpu 1.0 \
  --memory 2.0Gi \
  --registry-server $ACR_NAME.azurecr.io \
  --user-assigned $IDENTITY_ID \
  --env-vars \
    "SPRING_PROFILES_ACTIVE=azure" \
    "SPRING_DATASOURCE_URL=jdbc:postgresql://${PSQL_FQDN}:5432/${PSQL_DATABASE_NAME}" \
    "SPRING_DATASOURCE_USERNAME=${PSQL_ADMIN_USER}" \
    "AZURE_STORAGE_ACCOUNT_NAME=${STORAGE_ACCOUNT_NAME}" \
    "AZURE_SERVICEBUS_NAMESPACE=${SERVICEBUS_NAMESPACE}.servicebus.windows.net" \
    "AZURE_CLIENT_ID=${IDENTITY_CLIENT_ID}" \
    "APPLICATIONINSIGHTS_INSTRUMENTATION_KEY=${INSTRUMENTATION_KEY}"
```

#### Step 6: Deploy Worker Container App
```bash
az containerapp create \
  --name assetmanager-worker \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --image $ACR_NAME.azurecr.io/assetmanager-worker:latest \
  --min-replicas 1 \
  --max-replicas 5 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --registry-server $ACR_NAME.azurecr.io \
  --user-assigned $IDENTITY_ID \
  --env-vars \
    "SPRING_PROFILES_ACTIVE=azure" \
    "SPRING_DATASOURCE_URL=jdbc:postgresql://${PSQL_FQDN}:5432/${PSQL_DATABASE_NAME}" \
    "SPRING_DATASOURCE_USERNAME=${PSQL_ADMIN_USER}" \
    "AZURE_STORAGE_ACCOUNT_NAME=${STORAGE_ACCOUNT_NAME}" \
    "AZURE_SERVICEBUS_NAMESPACE=${SERVICEBUS_NAMESPACE}.servicebus.windows.net" \
    "AZURE_CLIENT_ID=${IDENTITY_CLIENT_ID}" \
    "APPLICATIONINSIGHTS_INSTRUMENTATION_KEY=${INSTRUMENTATION_KEY}"
```

### Todo Web API Application

#### Step 1: Build and Deploy
```bash
cd todo-web-api-use-oracle-db

# Build
mvn clean package -DskipTests

# Build Docker image
docker build -t todo-api:latest .

# Tag and push
docker tag todo-api:latest $ACR_NAME.azurecr.io/todo-api:latest
docker push $ACR_NAME.azurecr.io/todo-api:latest

# Deploy
az containerapp create \
  --name todo-api \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --image $ACR_NAME.azurecr.io/todo-api:latest \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 5 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --registry-server $ACR_NAME.azurecr.io \
  --user-assigned $IDENTITY_ID \
  --env-vars \
    "SPRING_PROFILES_ACTIVE=azure" \
    "SPRING_DATASOURCE_URL=jdbc:postgresql://${PSQL_FQDN}:5432/tododb" \
    "SPRING_DATASOURCE_USERNAME=${PSQL_ADMIN_USER}"
```

## Post-Deployment Validation

### Health Check Validation
```bash
# Get web app URL
WEB_APP_URL=$(az containerapp show \
  --name assetmanager-web \
  --resource-group $RESOURCE_GROUP \
  --query properties.configuration.ingress.fqdn -o tsv)

# Check health endpoint
curl https://$WEB_APP_URL/actuator/health

# Expected response:
# {"status":"UP"}
```

### Database Connectivity Test
```bash
# Connect to PostgreSQL
az postgres flexible-server connect \
  --name $PSQL_SERVER_NAME \
  --admin-user $PSQL_ADMIN_USER \
  --database-name $PSQL_DATABASE_NAME

# Run test query
SELECT version();
```

### Storage Access Test
```bash
# List containers
az storage container list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login

# Upload test file
echo "Test content" > test.txt
az storage blob upload \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name images \
  --file test.txt \
  --name test.txt \
  --auth-mode login
```

### Service Bus Test
```bash
# Send test message
az servicebus queue send \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $SERVICEBUS_NAMESPACE \
  --name $QUEUE_NAME \
  --body "Test message"
```

### Application Logs
```bash
# Stream web app logs
az containerapp logs show \
  --name assetmanager-web \
  --resource-group $RESOURCE_GROUP \
  --follow

# Stream worker app logs
az containerapp logs show \
  --name assetmanager-worker \
  --resource-group $RESOURCE_GROUP \
  --follow
```

## Monitoring & Alerting Setup

### Application Insights Queries
```kusto
// Failed requests
requests
| where success == false
| summarize count() by resultCode, name
| order by count_ desc

// Response times
requests
| summarize percentiles(duration, 50, 95, 99) by bin(timestamp, 5m)

// Exceptions
exceptions
| summarize count() by type, outerMessage
| order by count_ desc
```

### Create Alerts
```bash
# Alert for high error rate
az monitor metrics alert create \
  --name "High Error Rate" \
  --resource-group $RESOURCE_GROUP \
  --scopes "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.App/containerApps/assetmanager-web" \
  --condition "avg requests/failed > 10" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action email user@example.com

# Alert for high response time
az monitor metrics alert create \
  --name "High Response Time" \
  --resource-group $RESOURCE_GROUP \
  --scopes "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.App/containerApps/assetmanager-web" \
  --condition "avg requests/duration > 1000" \
  --window-size 5m \
  --evaluation-frequency 1m
```

## Scaling Configuration

### Manual Scaling
```bash
az containerapp update \
  --name assetmanager-web \
  --resource-group $RESOURCE_GROUP \
  --min-replicas 2 \
  --max-replicas 20
```

### Auto-scaling Rules
```bash
# Scale based on HTTP requests
az containerapp update \
  --name assetmanager-web \
  --resource-group $RESOURCE_GROUP \
  --scale-rule-name http-requests \
  --scale-rule-type http \
  --scale-rule-http-concurrency 100
```

## Backup & Disaster Recovery

### Database Backup
```bash
# Configure automated backups (already enabled by default)
az postgres flexible-server update \
  --resource-group $RESOURCE_GROUP \
  --name $PSQL_SERVER_NAME \
  --backup-retention 30 \
  --geo-redundant-backup Enabled
```

### Storage Backup
```bash
# Enable soft delete
az storage account blob-service-properties update \
  --account-name $STORAGE_ACCOUNT_NAME \
  --enable-delete-retention true \
  --delete-retention-days 30
```

## Troubleshooting

### Common Issues

#### Issue 1: Container App Not Starting
```bash
# Check logs
az containerapp logs show \
  --name assetmanager-web \
  --resource-group $RESOURCE_GROUP \
  --tail 100

# Check revision
az containerapp revision list \
  --name assetmanager-web \
  --resource-group $RESOURCE_GROUP
```

#### Issue 2: Database Connection Failed
```bash
# Verify firewall rules
az postgres flexible-server firewall-rule list \
  --resource-group $RESOURCE_GROUP \
  --name $PSQL_SERVER_NAME

# Test connection from local
psql "host=${PSQL_FQDN} port=5432 dbname=${PSQL_DATABASE_NAME} user=${PSQL_ADMIN_USER} sslmode=require"
```

#### Issue 3: Managed Identity Not Working
```bash
# Verify role assignments
az role assignment list \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --all

# Check identity assignment on container app
az containerapp show \
  --name assetmanager-web \
  --resource-group $RESOURCE_GROUP \
  --query identity
```

## Cleanup

### Delete All Resources
```bash
# Delete resource group (removes all resources)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait
```

### Delete Individual Resources
```bash
# Delete container apps
az containerapp delete --name assetmanager-web --resource-group $RESOURCE_GROUP --yes
az containerapp delete --name assetmanager-worker --resource-group $RESOURCE_GROUP --yes

# Delete other resources as needed
az postgres flexible-server delete --name $PSQL_SERVER_NAME --resource-group $RESOURCE_GROUP --yes
az storage account delete --name $STORAGE_ACCOUNT_NAME --resource-group $RESOURCE_GROUP --yes
az servicebus namespace delete --name $SERVICEBUS_NAMESPACE --resource-group $RESOURCE_GROUP
```

## Next Steps

1. **Configure Custom Domain**: Add custom domain and SSL certificate
2. **Set Up CI/CD**: Implement automated deployments with GitHub Actions
3. **Enable Private Endpoints**: Secure network access to Azure services
4. **Implement WAF**: Add Web Application Firewall for security
5. **Cost Optimization**: Review and optimize resource usage

## References

- [Azure Container Apps Documentation](https://docs.microsoft.com/azure/container-apps/)
- [Azure Database for PostgreSQL Documentation](https://docs.microsoft.com/azure/postgresql/)
- [Azure Managed Identity Documentation](https://docs.microsoft.com/azure/active-directory/managed-identities-azure-resources/)
- [Azure CLI Reference](https://docs.microsoft.com/cli/azure/)
- [Bicep Documentation](https://docs.microsoft.com/azure/azure-resource-manager/bicep/)

---

**Last Updated**: January 16, 2026  
**Version**: 1.0
