# CI/CD Pipeline Guide for Azure Deployment

## Overview

This guide provides templates and best practices for implementing CI/CD pipelines using GitHub Actions to automate the build, test, and deployment of Java applications to Azure.

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Repository                         │
├─────────────────────────────────────────────────────────────┤
│  Code Push/PR → GitHub Actions Workflows                    │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Build &   │→ │  Security    │→ │  Container   │      │
│  │   Test      │  │  Scan        │  │  Build       │      │
│  └─────────────┘  └──────────────┘  └──────────────┘      │
│                           │                  │              │
│                           ▼                  ▼              │
│                  ┌──────────────┐  ┌──────────────┐       │
│                  │  CodeQL      │  │  Push to ACR │       │
│                  │  Analysis    │  └──────────────┘       │
│                  └──────────────┘           │              │
│                                             ▼              │
│                           ┌──────────────────────┐        │
│                           │  Deploy to Azure     │        │
│                           │  Container Apps      │        │
│                           └──────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

## GitHub Secrets Configuration

### Required Secrets

Configure these secrets in your GitHub repository:
`Settings → Secrets and variables → Actions → New repository secret`

```
AZURE_CREDENTIALS              # Azure Service Principal credentials (JSON)
AZURE_SUBSCRIPTION_ID          # Azure subscription ID
AZURE_RESOURCE_GROUP          # Target resource group name
AZURE_CONTAINER_REGISTRY      # Azure Container Registry name
AZURE_CONTAINERAPPS_ENV       # Container Apps environment name
DB_PASSWORD                   # PostgreSQL database password (if not using Managed Identity)
```

### Creating Azure Service Principal

```bash
# Create service principal with contributor role
az ad sp create-for-rbac \
  --name "github-actions-java-migration" \
  --role contributor \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP \
  --sdk-auth

# Output format (save as AZURE_CREDENTIALS secret):
{
  "clientId": "<client-id>",
  "clientSecret": "<client-secret>",
  "subscriptionId": "<subscription-id>",
  "tenantId": "<tenant-id>",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```

## Workflow Templates

### 1. Build and Test Workflow

Create `.github/workflows/build-test.yml`:

```yaml
name: Build and Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

env:
  JAVA_VERSION: '21'
  MAVEN_OPTS: '-Xmx3072m'

jobs:
  build-test:
    name: Build and Test Java Applications
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        application:
          - name: asset-manager-web
            path: asset-manager/web
          - name: asset-manager-worker
            path: asset-manager/worker
          - name: todo-web-api
            path: todo-web-api-use-oracle-db
          - name: mi-sql-demo
            path: mi-sql-public-demo
          - name: rabbitmq-sender
            path: rabbitmq-sender
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up JDK ${{ env.JAVA_VERSION }}
      uses: actions/setup-java@v4
      with:
        java-version: ${{ env.JAVA_VERSION }}
        distribution: 'temurin'
        cache: maven
    
    - name: Build with Maven
      working-directory: ${{ matrix.application.path }}
      run: |
        mvn clean install -DskipTests
        mvn --version
    
    - name: Run unit tests
      working-directory: ${{ matrix.application.path }}
      run: mvn test
    
    - name: Run integration tests
      working-directory: ${{ matrix.application.path }}
      run: mvn verify -DskipUnitTests
      continue-on-error: true
    
    - name: Generate test report
      uses: dorny/test-reporter@v1
      if: success() || failure()
      with:
        name: Test Results - ${{ matrix.application.name }}
        path: ${{ matrix.application.path }}/target/surefire-reports/*.xml
        reporter: java-junit
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: ${{ matrix.application.path }}/target/site/jacoco/jacoco.xml
        flags: ${{ matrix.application.name }}
        name: codecov-${{ matrix.application.name }}
    
    - name: Upload build artifacts
      uses: actions/upload-artifact@v3
      with:
        name: ${{ matrix.application.name }}-jar
        path: ${{ matrix.application.path }}/target/*.jar
        retention-days: 7
```

### 2. Security Scanning Workflow

Create `.github/workflows/security-scan.yml`:

```yaml
name: Security Scanning

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 0 * * 0'  # Weekly on Sunday

jobs:
  codeql-analysis:
    name: CodeQL Analysis
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    
    strategy:
      matrix:
        language: [ 'java' ]
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Initialize CodeQL
      uses: github/codeql-action/init@v3
      with:
        languages: ${{ matrix.language }}
    
    - name: Set up JDK 21
      uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
        cache: maven
    
    - name: Build applications
      run: |
        cd asset-manager && mvn clean package -DskipTests
        cd ../todo-web-api-use-oracle-db && mvn clean package -DskipTests
        cd ../mi-sql-public-demo && mvn clean package -DskipTests
    
    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v3
  
  dependency-scan:
    name: Dependency Vulnerability Scan
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Run OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'java-migration-copilot-samples'
        path: '.'
        format: 'HTML'
        args: >
          --scan '**/pom.xml'
          --enableRetired
          --failOnCVSS 7
    
    - name: Upload Dependency Check Report
      uses: actions/upload-artifact@v3
      with:
        name: dependency-check-report
        path: reports/
  
  trivy-scan:
    name: Trivy Container Scan
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: 'trivy-results.sarif'
```

### 3. Container Build and Push Workflow

Create `.github/workflows/container-build.yml`:

```yaml
name: Build and Push Containers

on:
  push:
    branches: [ main ]
    paths:
      - 'asset-manager/**'
      - 'todo-web-api-use-oracle-db/**'
      - '.github/workflows/container-build.yml'
  workflow_dispatch:

env:
  REGISTRY: ${{ secrets.AZURE_CONTAINER_REGISTRY }}.azurecr.io
  JAVA_VERSION: '21'

jobs:
  build-and-push:
    name: Build and Push Container Images
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        include:
          - app: asset-manager-web
            context: ./asset-manager/web
            dockerfile: ./asset-manager/web/Dockerfile
          - app: asset-manager-worker
            context: ./asset-manager/worker
            dockerfile: ./asset-manager/worker/Dockerfile
          - app: todo-api
            context: ./todo-web-api-use-oracle-db
            dockerfile: ./todo-web-api-use-oracle-db/Dockerfile
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up JDK ${{ env.JAVA_VERSION }}
      uses: actions/setup-java@v4
      with:
        java-version: ${{ env.JAVA_VERSION }}
        distribution: 'temurin'
        cache: maven
    
    - name: Build application
      run: |
        cd ${{ matrix.context }}
        mvn clean package -DskipTests
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Azure Container Registry
      uses: azure/docker-login@v1
      with:
        login-server: ${{ env.REGISTRY }}
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ matrix.app }}
        tags: |
          type=ref,event=branch
          type=sha,prefix={{branch}}-
          type=semver,pattern={{version}}
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: ${{ matrix.context }}
        file: ${{ matrix.dockerfile }}
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ matrix.app }}:buildcache
        cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ matrix.app }}:buildcache,mode=max
    
    - name: Scan image with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ matrix.app }}:latest
        format: 'sarif'
        output: 'trivy-results-${{ matrix.app }}.sarif'
    
    - name: Upload Trivy results
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: 'trivy-results-${{ matrix.app }}.sarif'
```

### 4. Azure Deployment Workflow - Development

Create `.github/workflows/deploy-dev.yml`:

```yaml
name: Deploy to Development

on:
  push:
    branches: [ develop ]
  workflow_dispatch:

env:
  AZURE_RESOURCE_GROUP: ${{ secrets.AZURE_RESOURCE_GROUP_DEV }}
  AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  CONTAINER_APPS_ENVIRONMENT: ${{ secrets.AZURE_CONTAINERAPPS_ENV_DEV }}
  REGISTRY: ${{ secrets.AZURE_CONTAINER_REGISTRY }}.azurecr.io

jobs:
  deploy-infrastructure:
    name: Deploy Infrastructure
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy Bicep template
      uses: azure/arm-deploy@v1
      with:
        subscriptionId: ${{ env.AZURE_SUBSCRIPTION_ID }}
        resourceGroupName: ${{ env.AZURE_RESOURCE_GROUP }}
        template: ./infrastructure/main.bicep
        parameters: environment=dev
        failOnStdErr: false
    
    - name: Get deployment outputs
      id: outputs
      run: |
        ACR_NAME=$(az deployment group show \
          --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
          --name main \
          --query properties.outputs.acrLoginServer.value -o tsv)
        echo "acr-name=$ACR_NAME" >> $GITHUB_OUTPUT
  
  deploy-applications:
    name: Deploy Applications
    needs: deploy-infrastructure
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        app:
          - name: assetmanager-web
            image: asset-manager-web
            port: 8080
            ingress: external
          - name: assetmanager-worker
            image: asset-manager-worker
            port: 8080
            ingress: internal
          - name: todo-api
            image: todo-api
            port: 8080
            ingress: external
    
    steps:
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy to Container Apps
      uses: azure/container-apps-deploy-action@v1
      with:
        resourceGroup: ${{ env.AZURE_RESOURCE_GROUP }}
        containerAppName: ${{ matrix.app.name }}
        containerAppEnvironment: ${{ env.CONTAINER_APPS_ENVIRONMENT }}
        imageToDeploy: ${{ env.REGISTRY }}/${{ matrix.app.image }}:latest
        targetPort: ${{ matrix.app.port }}
        ingress: ${{ matrix.app.ingress }}
    
    - name: Get application URL
      id: app-url
      run: |
        APP_URL=$(az containerapp show \
          --name ${{ matrix.app.name }} \
          --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
          --query properties.configuration.ingress.fqdn -o tsv)
        echo "url=https://$APP_URL" >> $GITHUB_OUTPUT
    
    - name: Health check
      run: |
        sleep 30
        curl -f ${{ steps.app-url.outputs.url }}/actuator/health || exit 1
    
    - name: Comment PR with deployment URL
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v6
      with:
        script: |
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: '🚀 Deployed ${{ matrix.app.name }} to DEV: ${{ steps.app-url.outputs.url }}'
          })
```

### 5. Azure Deployment Workflow - Production

Create `.github/workflows/deploy-prod.yml`:

```yaml
name: Deploy to Production

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      image-tag:
        description: 'Docker image tag to deploy'
        required: true
        default: 'latest'

env:
  AZURE_RESOURCE_GROUP: ${{ secrets.AZURE_RESOURCE_GROUP_PROD }}
  AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  CONTAINER_APPS_ENVIRONMENT: ${{ secrets.AZURE_CONTAINERAPPS_ENV_PROD }}
  REGISTRY: ${{ secrets.AZURE_CONTAINER_REGISTRY }}.azurecr.io

jobs:
  approval:
    name: Manual Approval
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
    - name: Wait for approval
      run: echo "Deployment approved"
  
  deploy-production:
    name: Deploy to Production
    needs: approval
    runs-on: ubuntu-latest
    
    strategy:
      max-parallel: 1  # Deploy one at a time
      matrix:
        app:
          - name: assetmanager-web
            image: asset-manager-web
          - name: assetmanager-worker
            image: asset-manager-worker
          - name: todo-api
            image: todo-api
    
    steps:
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Get current revision
      id: current
      run: |
        CURRENT_REVISION=$(az containerapp revision list \
          --name ${{ matrix.app.name }} \
          --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
          --query "[0].name" -o tsv)
        echo "revision=$CURRENT_REVISION" >> $GITHUB_OUTPUT
    
    - name: Deploy new revision
      uses: azure/container-apps-deploy-action@v1
      with:
        resourceGroup: ${{ env.AZURE_RESOURCE_GROUP }}
        containerAppName: ${{ matrix.app.name }}
        imageToDeploy: ${{ env.REGISTRY }}/${{ matrix.app.image }}:${{ github.event.inputs.image-tag || 'latest' }}
    
    - name: Traffic split test (10%)
      run: |
        az containerapp ingress traffic set \
          --name ${{ matrix.app.name }} \
          --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
          --revision-weight latest=10 ${{ steps.current.outputs.revision }}=90
    
    - name: Monitor new revision
      run: |
        sleep 300  # Monitor for 5 minutes
        
        ERROR_COUNT=$(az monitor metrics list \
          --resource /subscriptions/${{ env.AZURE_SUBSCRIPTION_ID }}/resourceGroups/${{ env.AZURE_RESOURCE_GROUP }}/providers/Microsoft.App/containerApps/${{ matrix.app.name }} \
          --metric "Requests" \
          --filter "StatusCode eq '5*'" \
          --aggregation count \
          --query value[0].timeseries[0].data[-1].count -o tsv)
        
        if [ "$ERROR_COUNT" -gt 10 ]; then
          echo "High error rate detected, rolling back"
          exit 1
        fi
    
    - name: Full traffic cutover
      run: |
        az containerapp ingress traffic set \
          --name ${{ matrix.app.name }} \
          --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
          --revision-weight latest=100
    
    - name: Health check
      run: |
        APP_URL=$(az containerapp show \
          --name ${{ matrix.app.name }} \
          --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
          --query properties.configuration.ingress.fqdn -o tsv)
        
        curl -f https://$APP_URL/actuator/health || exit 1
    
    - name: Deactivate old revision
      run: |
        az containerapp revision deactivate \
          --name ${{ matrix.app.name }} \
          --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
          --revision ${{ steps.current.outputs.revision }}
  
  rollback:
    name: Rollback on Failure
    needs: deploy-production
    if: failure()
    runs-on: ubuntu-latest
    
    steps:
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Rollback deployment
      run: |
        for APP in assetmanager-web assetmanager-worker todo-api; do
          echo "Rolling back $APP"
          
          # Get previous stable revision
          STABLE_REVISION=$(az containerapp revision list \
            --name $APP \
            --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
            --query "[?properties.trafficWeight > 0] | [1].name" -o tsv)
          
          # Route all traffic to stable revision
          az containerapp ingress traffic set \
            --name $APP \
            --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
            --revision-weight $STABLE_REVISION=100
        done
    
    - name: Notify team
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        text: 'Production deployment failed and rolled back'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### 6. Database Migration Workflow

Create `.github/workflows/db-migration.yml`:

```yaml
name: Database Migration

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod

jobs:
  migrate:
    name: Run Database Migration
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up JDK 21
      uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
    
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Get database connection
      id: db
      run: |
        PSQL_HOST=$(az postgres flexible-server show \
          --name javamigration-${{ github.event.inputs.environment }}-psql \
          --resource-group ${{ secrets.AZURE_RESOURCE_GROUP }} \
          --query fullyQualifiedDomainName -o tsv)
        echo "host=$PSQL_HOST" >> $GITHUB_OUTPUT
    
    - name: Run Flyway migrations
      run: |
        cd asset-manager
        mvn flyway:migrate \
          -Dflyway.url=jdbc:postgresql://${{ steps.db.outputs.host }}:5432/assetmanager \
          -Dflyway.user=${{ secrets.DB_USER }} \
          -Dflyway.password=${{ secrets.DB_PASSWORD }}
```

## Best Practices

### 1. Environment-Specific Configurations

Use GitHub Environments for environment-specific secrets and protection rules:

- **Development**: Auto-deploy on push to develop branch
- **Staging**: Auto-deploy on push to main branch
- **Production**: Require manual approval before deployment

### 2. Branch Protection Rules

Configure branch protection for `main` and `develop`:
- Require pull request reviews
- Require status checks to pass
- Require branches to be up to date
- Include administrators

### 3. Workflow Optimization

```yaml
# Use caching to speed up builds
- name: Cache Maven packages
  uses: actions/cache@v3
  with:
    path: ~/.m2
    key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
    restore-keys: ${{ runner.os }}-m2

# Use matrix strategy for parallel execution
strategy:
  matrix:
    java-version: [17, 21]
  fail-fast: false
```

### 4. Notifications

Add Slack or Teams notifications:

```yaml
- name: Notify deployment status
  uses: 8398a7/action-slack@v3
  if: always()
  with:
    status: ${{ job.status }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
    fields: repo,message,commit,author,action,eventName,ref,workflow
```

## Monitoring & Alerting

### Application Insights Integration

Add Application Insights to monitor deployments:

```yaml
- name: Track deployment in Application Insights
  run: |
    curl -X POST https://dc.services.visualstudio.com/v2/track \
      -H "Content-Type: application/json" \
      -d '{
        "name": "Microsoft.ApplicationInsights.Event",
        "time": "'$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)'",
        "iKey": "${{ secrets.APPINSIGHTS_KEY }}",
        "data": {
          "baseType": "EventData",
          "baseData": {
            "name": "Deployment",
            "properties": {
              "Environment": "${{ github.event.inputs.environment }}",
              "Version": "${{ github.sha }}",
              "Status": "Success"
            }
          }
        }
      }'
```

## Troubleshooting

### Common Issues

1. **Authentication failures**: Verify service principal has correct permissions
2. **Container registry access**: Ensure ACR has appropriate firewall rules
3. **Deployment timeouts**: Increase timeout values or optimize container size
4. **Failed health checks**: Verify application starts correctly and health endpoint is accessible

## Next Steps

1. Implement the workflows in your repository
2. Configure required secrets
3. Test in development environment
4. Gradually roll out to staging and production
5. Monitor and optimize based on metrics

---

**Last Updated**: January 16, 2026  
**Version**: 1.0
