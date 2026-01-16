# Quick Start Guide: Modernize Your Java Application for Azure

This guide provides a fast-track approach to modernizing your Java application and deploying it to Azure using GitHub Copilot app modernization.

## ⚡ 30-Minute Quick Start

### Prerequisites Check (5 minutes)

```bash
# Check your tools
java --version          # Need: Java 17 or 21
mvn --version          # Need: Maven 3.8+
docker --version       # Need: Docker 20.10+
az --version           # Need: Azure CLI 2.50+
git --version          # Need: Git

# Login to Azure
az login
az account set --subscription "YOUR_SUBSCRIPTION_ID"
```

### Choose Your Path (1 minute)

Pick the sample that best matches your migration scenario:

| Your Situation | Sample to Use | Time Required |
|----------------|--------------|---------------|
| Need to secure database connections | [mi-sql-public-demo](mi-sql-public-demo) | 2-4 hours |
| Using Oracle Database | [todo-web-api-use-oracle-db](todo-web-api-use-oracle-db) | 1-2 days |
| Using AWS S3 and RabbitMQ | [asset-manager](asset-manager) | 2-3 days |
| Legacy Java EE application | [jakarta-ee/student-web-app](jakarta-ee/student-web-app) | 2-3 days |

### Run Sample Locally (10 minutes)

**For Asset Manager:**
```bash
cd asset-manager

# Start dependencies (PostgreSQL, RabbitMQ)
docker-compose up -d

# Build and run web application
cd web
mvn clean package -DskipTests
mvn spring-boot:run

# In another terminal, run worker
cd ../worker
mvn clean package -DskipTests
mvn spring-boot:run

# Access at http://localhost:8080
```

**For Todo Web API:**
```bash
cd todo-web-api-use-oracle-db

# Note: Requires Oracle Database. See README for setup.
mvn clean package
mvn spring-boot:run

# Test API at http://localhost:8080
curl http://localhost:8080/api/todos
```

**For MI SQL Demo:**
```bash
cd mi-sql-public-demo

# Update src/main/resources/application.properties with your DB
mvn clean package
java -jar target/demo-1.0-SNAPSHOT.jar
```

### Deploy to Azure (15 minutes)

**Option 1: Quick Deploy with Azure CLI**
```bash
# Set variables
RESOURCE_GROUP="java-migration-demo-rg"
LOCATION="eastus"
APP_NAME="my-java-app"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create Container Apps environment
az containerapp env create \
  --name "${APP_NAME}-env" \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Build and deploy (from your project directory)
az containerapp up \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --environment "${APP_NAME}-env" \
  --source .
```

**Option 2: Use GitHub Copilot App Modernization**
1. Open your project in VS Code
2. Install "GitHub Copilot app modernization" extension
3. Click "Migrate to Azure" in the sidebar
4. Follow the guided workflow

## 📖 Detailed Guides by Scenario

### Scenario 1: Secure Database Connections (Easiest)

**Goal**: Replace username/password with Azure Managed Identity

**Steps**:
1. Open [mi-sql-public-demo](mi-sql-public-demo)
2. Review current connection code
3. Follow the Managed Identity migration guide
4. Deploy to Azure

**Key Changes**:
```java
// Before
String connectionUrl = "jdbc:sqlserver://myserver.database.windows.net:1433;" +
                       "database=mydb;user=admin;password=MyPassword123;";

// After (with Managed Identity)
String connectionUrl = "jdbc:sqlserver://myserver.database.windows.net:1433;" +
                       "database=mydb;authentication=ActiveDirectoryMSI;";
```

**Time**: 2-4 hours

---

### Scenario 2: Migrate from Oracle to PostgreSQL

**Goal**: Replace Oracle Database with Azure Database for PostgreSQL

**Steps**:
1. Open [todo-web-api-use-oracle-db](todo-web-api-use-oracle-db)
2. Analyze Oracle-specific features used
3. Use GitHub Copilot to suggest PostgreSQL equivalents
4. Update dependencies and configuration
5. Test and deploy

**Key Changes**:
```xml
<!-- Before -->
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc11</artifactId>
</dependency>

<!-- After -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

```java
// Before (Oracle-specific)
@Column(columnDefinition = "VARCHAR2(255)")
private String title;

// After (PostgreSQL)
@Column(length = 255)
private String title;
```

**Time**: 1-2 days

---

### Scenario 3: Complete Cloud Migration (Most Comprehensive)

**Goal**: Migrate multi-service app from AWS/on-premises to Azure

**Steps**:
1. Open [asset-manager](asset-manager)
2. Follow the complete workshop guide
3. Upgrade Java and Spring Boot versions
4. Migrate AWS S3 → Azure Blob Storage
5. Migrate RabbitMQ → Azure Service Bus
6. Containerize and deploy

**Key Changes**:
```java
// Before (AWS S3)
import software.amazon.awssdk.services.s3.S3Client;

S3Client s3 = S3Client.create();
s3.putObject(request, RequestBody.fromFile(file));

// After (Azure Blob Storage)
import com.azure.storage.blob.BlobServiceClient;

BlobServiceClient blobService = new BlobServiceClientBuilder()
    .credential(new DefaultAzureCredentialBuilder().build())
    .buildClient();
blobService.getBlobContainerClient("images")
    .getBlobClient("file.jpg")
    .uploadFromFile(file.getPath());
```

**Time**: 2-3 days

---

### Scenario 4: Modernize Legacy Java EE Application

**Goal**: Update Java EE app to Jakarta EE and migrate to Azure

**Steps**:
1. Open [jakarta-ee/student-web-app](jakarta-ee/student-web-app)
2. Migrate from Ant to Maven
3. Update javax.* imports to jakarta.*
4. Modernize database connections
5. Containerize and deploy

**Key Changes**:
```java
// Before (Java EE)
import javax.servlet.http.HttpServlet;
import javax.persistence.Entity;

// After (Jakarta EE)
import jakarta.servlet.http.HttpServlet;
import jakarta.persistence.Entity;
```

**Time**: 2-3 days

## 🎯 Common Migration Patterns

### Pattern 1: Add Spring Boot Actuator for Health Checks

```xml
<!-- Add to pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# Add to application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always
```

Access health endpoint: `http://localhost:8080/actuator/health`

### Pattern 2: Implement Azure Managed Identity

```xml
<!-- Add to pom.xml -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-identity</artifactId>
    <version>1.11.0</version>
</dependency>
```

```java
// Use in your code
import com.azure.identity.DefaultAzureCredentialBuilder;

DefaultAzureCredential credential = new DefaultAzureCredentialBuilder().build();
```

### Pattern 3: Containerize with Docker

Create `Dockerfile`:
```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build and run:
```bash
mvn clean package
docker build -t my-app:latest .
docker run -p 8080:8080 my-app:latest
```

### Pattern 4: Add Application Insights

```xml
<!-- Add to pom.xml -->
<dependency>
    <groupId>com.azure.spring</groupId>
    <artifactId>spring-cloud-azure-starter-monitor</artifactId>
    <version>5.14.0</version>
</dependency>
```

```yaml
# Add to application.yml
azure:
  application-insights:
    instrumentation-key: ${APPLICATIONINSIGHTS_INSTRUMENTATION_KEY}
```

## 🚀 Deployment Options

### Option 1: Azure Container Apps (Recommended)
**Best for**: Microservices, containerized apps, auto-scaling needs

```bash
az containerapp up \
  --name my-app \
  --resource-group my-rg \
  --location eastus \
  --source . \
  --ingress external \
  --target-port 8080
```

**Pros**: Serverless, auto-scaling, no infrastructure management  
**Cost**: Pay per use, starts at ~$50/month

### Option 2: Azure App Service
**Best for**: Traditional web apps, simple deployment

```bash
az webapp up \
  --name my-app \
  --resource-group my-rg \
  --runtime "JAVA:21-java21" \
  --sku B1
```

**Pros**: Easy to use, built-in deployment slots  
**Cost**: Fixed pricing, starts at ~$13/month

### Option 3: Azure Kubernetes Service (AKS)
**Best for**: Complex microservices, need full K8s features

```bash
az aks create \
  --name my-cluster \
  --resource-group my-rg \
  --node-count 2 \
  --generate-ssh-keys

kubectl apply -f deployment.yaml
```

**Pros**: Full control, advanced features  
**Cost**: Higher, starts at ~$150/month

## ✅ Pre-Deployment Checklist

Before deploying to production:

- [ ] Update Java version to 17 or 21
- [ ] Update Spring Boot to 3.x (if applicable)
- [ ] Replace passwords with Managed Identity
- [ ] Add health check endpoints (`/actuator/health`)
- [ ] Configure Application Insights
- [ ] Add proper logging (JSON format recommended)
- [ ] Test locally with Docker
- [ ] Run security scans (CVE checks)
- [ ] Create deployment pipeline (GitHub Actions)
- [ ] Set up monitoring and alerts
- [ ] Document environment variables
- [ ] Create rollback plan

## 🔍 Troubleshooting

### Issue: Application won't start in container

**Solution**:
```bash
# Check logs
docker logs <container-id>

# Common fixes:
# 1. Verify Java version matches
java -version

# 2. Check application.properties
# Ensure all required properties are set

# 3. Verify dependencies
mvn dependency:tree
```

### Issue: Database connection fails

**Solution**:
```bash
# Test connection from local machine
az postgres flexible-server connect \
  --name myserver \
  --admin-user myadmin \
  --database-name mydb

# Check firewall rules
az postgres flexible-server firewall-rule list \
  --name myserver \
  --resource-group my-rg

# Add your IP if needed
az postgres flexible-server firewall-rule create \
  --name AllowMyIP \
  --resource-group my-rg \
  --server-name myserver \
  --start-ip-address YOUR_IP \
  --end-ip-address YOUR_IP
```

### Issue: Managed Identity not working

**Solution**:
```bash
# Check identity is assigned
az containerapp show \
  --name my-app \
  --resource-group my-rg \
  --query identity

# Verify role assignments
az role assignment list \
  --assignee <managed-identity-principal-id>

# Assign missing roles
az role assignment create \
  --assignee <managed-identity-principal-id> \
  --role "Storage Blob Data Contributor" \
  --scope <storage-account-id>
```

## 📊 Cost Optimization Tips

1. **Use consumption-based pricing**: Container Apps charges only for what you use
2. **Right-size your resources**: Start small and scale up as needed
3. **Use dev/test pricing**: Save up to 55% on non-production workloads
4. **Enable auto-scaling**: Scale down during off-hours
5. **Use Azure Reserved Instances**: Save up to 72% on predictable workloads
6. **Monitor and optimize**: Use Azure Advisor recommendations

**Example Cost Breakdown** (Small application):
```
Azure Container Apps:     $50-100/month
PostgreSQL (Standard):    $30-60/month
Blob Storage (10GB):      $1-2/month
Service Bus (Standard):   $10/month
Application Insights:     $20-40/month
-------------------------------------------
Total:                    $111-212/month
```

## 🎓 Next Steps

### Beginner
1. ✅ Complete this quick start guide
2. 📖 Read [Azure Service Mapping Reference](AZURE_SERVICE_MAPPING.md)
3. 🚀 Deploy your first sample to Azure
4. 📚 Review Azure best practices documentation

### Intermediate
1. 🏗️ Follow [Azure Modernization Plan](AZURE_MODERNIZATION_PLAN.md)
2. 🔄 Set up CI/CD with [CI/CD Pipeline Guide](CI_CD_PIPELINE_GUIDE.md)
3. 🔐 Implement Managed Identity across all services
4. 📊 Configure comprehensive monitoring

### Advanced
1. 🏛️ Design production-ready architecture
2. 🌍 Implement multi-region deployment
3. 🔒 Add advanced security (WAF, Private Link)
4. 💰 Optimize costs and performance

## 📚 Additional Resources

### Documentation
- [Azure Modernization Plan](AZURE_MODERNIZATION_PLAN.md) - Complete strategy
- [Azure Deployment Guide](AZURE_DEPLOYMENT_GUIDE.md) - Detailed deployment steps
- [CI/CD Pipeline Guide](CI_CD_PIPELINE_GUIDE.md) - Automation templates
- [Azure Service Mapping](AZURE_SERVICE_MAPPING.md) - Technology mappings

### Microsoft Learn
- [Java on Azure](https://docs.microsoft.com/learn/paths/java-on-azure/)
- [Migrate Java Applications to Azure](https://docs.microsoft.com/learn/modules/migrate-java-app-azure/)
- [Azure Container Apps](https://docs.microsoft.com/learn/paths/deploy-applications-azure-container-apps/)

### Tools
- [GitHub Copilot App Modernization](https://marketplace.visualstudio.com/items?itemName=vscjava.migrate-java-to-azure)
- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
- [Azure Portal](https://portal.azure.com)

## 💬 Get Help

- **GitHub Issues**: Report bugs or request features
- **GitHub Discussions**: Ask questions and share experiences
- **Microsoft Q&A**: Get help from Microsoft engineers
- **Stack Overflow**: Tag questions with `azure` and `java`

---

**Ready to start?** Pick a sample and follow the steps above! 🚀

**Questions?** Check our [FAQ](https://github.com/Azure-Samples/java-migration-copilot-samples/wiki/FAQ) or open a [Discussion](https://github.com/Azure-Samples/java-migration-copilot-samples/discussions)

**Need help?** We're here to support your migration journey! 💙
