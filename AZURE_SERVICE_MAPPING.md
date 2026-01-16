# Azure Service Mapping Reference

## Quick Reference: Technology to Azure Service Mapping

This document provides a quick reference for mapping current technologies to their Azure equivalents for the Java migration projects.

## Storage Services

### AWS S3 → Azure Blob Storage

| AWS S3 Concept | Azure Blob Storage Equivalent | Notes |
|----------------|------------------------------|-------|
| Bucket | Container | Top-level namespace for blobs |
| Object | Blob | Individual file/data |
| Object Key | Blob Name | Unique identifier within container |
| S3Client | BlobServiceClient | Main client class |
| PutObject | UploadBlob | Upload operation |
| GetObject | DownloadBlob | Download operation |
| DeleteObject | DeleteBlob | Delete operation |
| ListObjects | ListBlobs | List operation |
| Access Key/Secret | Managed Identity or Connection String | Authentication method |
| Multipart Upload | Block Blob Upload | Large file upload |
| Pre-signed URLs | SAS Tokens | Temporary access URLs |

**Migration Code Example:**

```java
// Before (AWS S3)
S3Client s3Client = S3Client.builder()
    .region(Region.US_EAST_1)
    .credentialsProvider(StaticCredentialsProvider.create(
        AwsBasicCredentials.create(accessKey, secretKey)))
    .build();

s3Client.putObject(PutObjectRequest.builder()
    .bucket("my-bucket")
    .key("my-object")
    .build(), 
    RequestBody.fromFile(file));

// After (Azure Blob Storage)
BlobServiceClient blobServiceClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(new DefaultAzureCredentialBuilder().build())
    .buildClient();

BlobContainerClient containerClient = blobServiceClient
    .getBlobContainerClient("my-container");
BlobClient blobClient = containerClient.getBlobClient("my-object");
blobClient.uploadFromFile(file.getPath());
```

### Local File System → Azure Blob Storage

| Local FS Operation | Azure Blob Storage | Implementation |
|-------------------|-------------------|----------------|
| File.createNewFile() | BlobClient.upload() | Upload blob |
| FileInputStream | BlobClient.openInputStream() | Read blob |
| FileOutputStream | BlobClient.openOutputStream() | Write blob |
| File.delete() | BlobClient.delete() | Delete blob |
| File.exists() | BlobClient.exists() | Check existence |
| File.listFiles() | ContainerClient.listBlobs() | List blobs |

## Messaging Services

### RabbitMQ → Azure Service Bus

| RabbitMQ Concept | Azure Service Bus Equivalent | Notes |
|------------------|----------------------------|-------|
| Exchange | Topic | Message routing |
| Queue | Queue | Message storage |
| Routing Key | Subscription Filter | Message filtering |
| Message | Message | Data payload |
| Connection | ServiceBusClient | Connection object |
| Channel | ServiceBusSender/Receiver | Communication channel |
| Publisher | ServiceBusSender | Message producer |
| Consumer | ServiceBusReceiver | Message consumer |
| Dead Letter Queue | Dead-letter sub-queue | Failed message handling |
| Message TTL | TimeToLive | Message expiration |
| Durable Queue | Persistent Queue | Survives restarts |
| Delayed Message | Scheduled Message | Delayed delivery |

**Migration Code Example:**

```java
// Before (RabbitMQ with Spring AMQP)
@Configuration
public class RabbitConfig {
    @Bean
    public Queue queue() {
        return new Queue("image-processing", true);
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        return new RabbitTemplate(connectionFactory);
    }
}

// After (Azure Service Bus with Spring Cloud Azure)
@Configuration
public class ServiceBusConfig {
    @Bean
    public ServiceBusProcessorClient serviceBusProcessor(
            ServiceBusClientBuilder builder) {
        return builder
            .processor()
            .queueName("image-processing")
            .processMessage(this::processMessage)
            .processError(this::processError)
            .buildProcessorClient();
    }
}
```

## Database Services

### Oracle Database → Azure Database for PostgreSQL

| Oracle Feature | PostgreSQL Equivalent | Migration Notes |
|----------------|----------------------|-----------------|
| VARCHAR2 | VARCHAR | Direct replacement |
| NUMBER | NUMERIC/DECIMAL | Adjust precision |
| SEQUENCE | SERIAL/SEQUENCE | Similar concept |
| SYSDATE | NOW() | Current timestamp |
| NVL() | COALESCE() | Null handling |
| DECODE() | CASE WHEN | Conditional logic |
| ROWNUM | LIMIT/OFFSET | Row limiting |
| CONNECT BY | WITH RECURSIVE | Hierarchical queries |
| DUAL table | SELECT without FROM | Not needed |
| MERGE | INSERT...ON CONFLICT | Upsert operation |
| PL/SQL | PL/pgSQL | Stored procedures |
| Oracle JDBC Driver | PostgreSQL JDBC Driver | ojdbc → postgresql |

**Migration Code Example:**

```java
// Before (Oracle)
@Entity
@Table(name = "TODO_ITEMS")
public class TodoItem {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, 
                    generator = "todo_seq")
    @SequenceGenerator(name = "todo_seq", 
                      sequenceName = "TODO_SEQ", 
                      allocationSize = 1)
    private Long id;
    
    @Column(name = "TITLE", columnDefinition = "VARCHAR2(255)")
    private String title;
}

// After (PostgreSQL)
@Entity
@Table(name = "todo_items")
public class TodoItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "title", length = 255)
    private String title;
}
```

**Common SQL Migrations:**

```sql
-- Oracle
SELECT * FROM (
    SELECT * FROM todo_items ORDER BY created_date DESC
) WHERE ROWNUM <= 10;

-- PostgreSQL
SELECT * FROM todo_items 
ORDER BY created_date DESC 
LIMIT 10;

-- Oracle
SELECT NVL(description, 'No description') FROM todo_items;

-- PostgreSQL
SELECT COALESCE(description, 'No description') FROM todo_items;

-- Oracle
SELECT DECODE(status, 'ACTIVE', 1, 'INACTIVE', 0, -1) FROM todo_items;

-- PostgreSQL
SELECT CASE status
    WHEN 'ACTIVE' THEN 1
    WHEN 'INACTIVE' THEN 0
    ELSE -1
END FROM todo_items;
```

### SQL Server (On-Premises) → Azure SQL Database

| Feature | Notes |
|---------|-------|
| Authentication | Migrate from SQL auth to Azure AD/Managed Identity |
| Connection String | Update to Azure SQL endpoint |
| Firewall | Configure Azure SQL firewall rules |
| Backup | Automated backups in Azure |
| High Availability | Built-in with Azure SQL |

## Authentication Services

### Password-Based → Azure Managed Identity

| Authentication Type | Before | After |
|-------------------|--------|-------|
| **Database** | Username/Password in config | Managed Identity with Azure AD |
| **Storage** | Access Key/Secret Key | Managed Identity or SAS Token |
| **Messaging** | Connection String with password | Managed Identity |
| **Key Vault** | Service Principal | Managed Identity |

**Migration Example:**

```java
// Before (Password-based PostgreSQL)
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.datasource.password=MySecretPassword123

// After (Managed Identity)
spring.datasource.url=jdbc:postgresql://myserver.postgres.database.azure.com:5432/mydb
spring.datasource.username=myManagedIdentity@myserver
spring.datasource.azure.passwordless-enabled=true
spring.cloud.azure.credential.managed-identity-enabled=true
```

```java
// Java code with Managed Identity
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://myserver.postgres.database.azure.com:5432/mydb")
            .username("myManagedIdentity@myserver")
            .build();
    }
}
```

## Compute Services

### On-Premises/VM → Azure Container Apps

| Aspect | Traditional Deployment | Azure Container Apps |
|--------|----------------------|---------------------|
| Infrastructure | Manual server setup | Fully managed |
| Scaling | Manual/basic auto-scaling | Automatic based on load |
| Deployment | Manual or custom scripts | Container-based |
| Load Balancing | External load balancer | Built-in |
| SSL/TLS | Manual certificate management | Managed certificates |
| Monitoring | Third-party tools | Application Insights |
| Cost Model | Fixed (24/7) | Consumption-based |

### Java EE Application Server → Azure App Service

| Java EE Server | Azure App Service Option | Notes |
|----------------|------------------------|-------|
| WebLogic | Azure App Service (Java SE) or AKS | Requires containerization |
| JBoss EAP | Azure App Service | Native support |
| Open Liberty | Azure Container Apps | Via container |
| Tomcat | Azure App Service | Native support |
| GlassFish | Azure Container Apps | Via container |

## Framework Migrations

### Spring Boot 2.x → Spring Boot 3.x

| Spring Boot 2.x | Spring Boot 3.x | Migration Action |
|----------------|----------------|------------------|
| javax.* packages | jakarta.* packages | Update all imports |
| Java 8 minimum | Java 17 minimum | Upgrade JDK |
| Spring Framework 5.x | Spring Framework 6.x | Update dependencies |
| Spring Security 5.x | Spring Security 6.x | Update security config |
| Hibernate 5.x | Hibernate 6.x | Update entity mappings |

### Java EE → Jakarta EE

| Java EE Package | Jakarta EE Package |
|----------------|-------------------|
| javax.servlet.* | jakarta.servlet.* |
| javax.persistence.* | jakarta.persistence.* |
| javax.transaction.* | jakarta.transaction.* |
| javax.annotation.* | jakarta.annotation.* |
| javax.ws.rs.* | jakarta.ws.rs.* |
| javax.validation.* | jakarta.validation.* |

## Monitoring & Observability

### Application Monitoring

| Aspect | Tool/Service | Azure Equivalent |
|--------|-------------|------------------|
| Metrics | Prometheus, Grafana | Azure Monitor, Application Insights |
| Logs | ELK Stack, Splunk | Azure Log Analytics |
| Tracing | Jaeger, Zipkin | Application Insights |
| APM | New Relic, Datadog | Application Insights |
| Alerts | PagerDuty | Azure Monitor Alerts |
| Dashboards | Custom dashboards | Azure Dashboards, Workbooks |

### Health Checks

```java
// Spring Boot Actuator (works with Azure Container Apps)
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection connection = dataSource.getConnection()) {
            if (connection.isValid(1000)) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .build();
            }
        } catch (SQLException e) {
            return Health.down()
                .withException(e)
                .build();
        }
        return Health.down().build();
    }
}
```

## Network & Security

### Network Services

| Feature | On-Premises | Azure |
|---------|------------|-------|
| VPN | Site-to-site VPN | Azure VPN Gateway |
| Private Network | VLAN | Virtual Network (VNet) |
| Load Balancer | Hardware LB | Azure Load Balancer |
| WAF | Third-party WAF | Azure Application Gateway WAF |
| DNS | Bind, Windows DNS | Azure DNS |
| CDN | Cloudflare, Akamai | Azure CDN |

### Security Services

| Security Aspect | Tool/Service | Azure Equivalent |
|----------------|-------------|------------------|
| Secrets Management | HashiCorp Vault | Azure Key Vault |
| Certificate Management | Let's Encrypt | Azure Key Vault, Managed Certificates |
| Identity & Access | LDAP, Active Directory | Azure Active Directory |
| Firewall | iptables, hardware firewall | Network Security Groups, Azure Firewall |
| DDoS Protection | Third-party | Azure DDoS Protection |
| Security Scanning | SonarQube, Snyk | Azure Security Center, GitHub Advanced Security |

## Development & CI/CD

### Build & Deploy

| Tool | Purpose | Azure Alternative |
|------|---------|------------------|
| Jenkins | CI/CD | Azure DevOps, GitHub Actions |
| GitLab CI | CI/CD | GitHub Actions |
| Ansible | Configuration | Azure Resource Manager, Bicep |
| Terraform | IaC | Azure Resource Manager, Bicep |
| Docker Registry | Container images | Azure Container Registry |
| Artifactory | Artifact storage | Azure Artifacts |

## Cost Comparison

### Rough Cost Estimates (USD/month)

| Service Category | On-Premises/AWS | Azure | Savings |
|-----------------|----------------|-------|---------|
| Compute (2 apps) | $200-400 | $100-200 | ~50% |
| Database (PostgreSQL) | $100-200 | $50-150 | ~30% |
| Storage (100GB) | $20-40 | $5-20 | ~60% |
| Messaging | $50-100 | $10-50 | ~50% |
| Monitoring | $100-200 | $50-100 | ~40% |
| **Total Estimated** | **$470-940** | **$215-520** | **~45%** |

*Note: Actual costs vary based on usage, region, and specific configurations. Use Azure Pricing Calculator for accurate estimates.*

## Performance Considerations

### Latency

| Connection Type | Typical Latency | Optimization |
|----------------|----------------|--------------|
| App → Database | <5ms (same region) | Use same Azure region |
| App → Storage | <10ms | Enable CDN for static content |
| App → Service Bus | <10ms | Use same region |
| Cross-Region | 50-200ms | Use Traffic Manager, replication |

### Throughput

| Service | Throughput | Scaling |
|---------|-----------|---------|
| Container Apps | 1000+ req/s per instance | Auto-scale |
| Storage Blob | 20,000+ req/s | Partitioning |
| Service Bus | 1000+ msg/s | Premium tier |
| PostgreSQL | 10,000+ IOPS | Scale up SKU |

## Migration Checklist

### Pre-Migration

- [ ] Inventory all applications and dependencies
- [ ] Assess Azure service compatibility
- [ ] Estimate costs with Azure Pricing Calculator
- [ ] Create Azure subscription and resource groups
- [ ] Set up development/test environments

### During Migration

- [ ] Update Java version (8 → 17/21)
- [ ] Update Spring Boot (2.x → 3.x)
- [ ] Migrate database (Oracle → PostgreSQL)
- [ ] Migrate storage (S3 → Blob Storage)
- [ ] Migrate messaging (RabbitMQ → Service Bus)
- [ ] Implement Managed Identity
- [ ] Add health check endpoints
- [ ] Create Dockerfiles
- [ ] Set up CI/CD pipelines
- [ ] Configure monitoring

### Post-Migration

- [ ] Performance testing
- [ ] Security scanning
- [ ] Cost optimization
- [ ] Documentation updates
- [ ] Team training
- [ ] Go-live preparation

## Support Resources

### Documentation

- [Azure Migration Guide](https://docs.microsoft.com/azure/cloud-adoption-framework/migrate/)
- [Spring Cloud Azure](https://spring.io/projects/spring-cloud-azure)
- [Azure SDK for Java](https://docs.microsoft.com/java/azure/)
- [Azure Architecture Center](https://docs.microsoft.com/azure/architecture/)

### Tools

- [Azure Migrate](https://azure.microsoft.com/services/azure-migrate/)
- [Azure TCO Calculator](https://azure.microsoft.com/pricing/tco/)
- [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/)
- [GitHub Copilot App Modernization](https://marketplace.visualstudio.com/items?itemName=vscjava.migrate-java-to-azure)

### Community

- [Microsoft Q&A - Java on Azure](https://docs.microsoft.com/answers/topics/azure-java-sdk.html)
- [Spring Cloud Azure GitHub](https://github.com/Azure/azure-sdk-for-java/tree/main/sdk/spring)
- [Azure Java Developers Slack](https://azurecommunity.slack.com)

---

**Last Updated**: January 16, 2026  
**Version**: 1.0  
**Maintained By**: Azure Migration Team
