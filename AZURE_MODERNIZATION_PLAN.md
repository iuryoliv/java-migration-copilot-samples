# Azure Modernization Plan for Java Migration Copilot Samples

## Executive Summary

This document provides a comprehensive plan to modernize and migrate the Java applications in this repository to Azure. The repository contains multiple sample applications demonstrating various migration scenarios from on-premises or other cloud platforms to Azure cloud services.

## Current State Analysis

### Java Applications Overview

| Application | Current Tech Stack | Java Version | Framework | Dependencies |
|-------------|-------------------|--------------|-----------|--------------|
| **asset-manager** | AWS S3, RabbitMQ, PostgreSQL | Java 8 | Spring Boot 2.7.18 | AWS SDK, Spring AMQP |
| **todo-web-api-use-oracle-db** | Oracle Database | Java 17 | Spring Boot 3.2.4 | Oracle JDBC |
| **mi-sql-public-demo** | SQL Server (password auth) | Java 17 | Standalone JDBC | MSSQL JDBC |
| **rabbitmq-sender** | RabbitMQ | Java 17 | Spring Boot 3.3.0 | Already migrated to Azure Service Bus |
| **jakarta-ee/student-web-app** | Java EE, Open Liberty | Java 8 (estimated) | Servlets + Spring MVC | MyBatis, Ant build |

### Non-Java Applications (Out of Scope)
- **ContosoUniversity** - C# .NET application
- **Malshinon** - C# .NET application

## Azure Modernization Strategy

### Phase 1: Assessment & Planning

#### 1.1 Application Assessment
For each Java application:
- Analyze dependencies and identify Azure equivalents
- Evaluate Java version compatibility with Azure services
- Assess Spring Boot/framework version requirements
- Identify security vulnerabilities (CVE checks)
- Document data persistence requirements
- Map messaging patterns to Azure services

#### 1.2 Target Azure Architecture

**Asset Manager Application:**
- **Compute**: Azure Container Apps (recommended) or Azure Kubernetes Service (AKS)
- **Storage**: Azure Blob Storage (replace AWS S3)
- **Messaging**: Azure Service Bus (replace RabbitMQ)
- **Database**: Azure Database for PostgreSQL Flexible Server
- **Authentication**: Azure Managed Identity (replace password-based auth)
- **Monitoring**: Azure Application Insights
- **Health Checks**: Spring Boot Actuator endpoints

**Todo Web API Application:**
- **Compute**: Azure App Service or Azure Container Apps
- **Database**: Azure Database for PostgreSQL Flexible Server (replace Oracle)
- **Authentication**: Azure Managed Identity
- **Monitoring**: Azure Application Insights

**MI SQL Public Demo:**
- **Compute**: Azure App Service or Azure Container Apps
- **Database**: Azure SQL Database or Azure SQL Managed Instance
- **Authentication**: Azure Managed Identity (replace password auth)

**Jakarta EE Student Web App:**
- **Compute**: Azure App Service with JBoss EAP or Azure Container Apps
- **Database**: Azure Database for PostgreSQL or MySQL
- **Build**: Migrate from Ant to Maven
- **Framework**: Migrate from Java EE to Jakarta EE 10

### Phase 2: Runtime & Framework Upgrades

#### 2.1 Java Version Upgrades

| Application | Current Version | Target Version | Priority |
|-------------|----------------|----------------|----------|
| asset-manager | Java 8 | Java 21 | High |
| jakarta-ee/student-web-app | Java 8 | Java 17 or 21 | High |
| todo-web-api-use-oracle-db | Java 17 | Java 21 (optional) | Low |
| mi-sql-public-demo | Java 17 | Java 21 (optional) | Low |
| rabbitmq-sender | Java 17 | Java 21 (optional) | Low |

#### 2.2 Framework Upgrades

| Application | Current Framework | Target Framework | Changes Required |
|-------------|------------------|------------------|------------------|
| asset-manager | Spring Boot 2.7.18 | Spring Boot 3.4.x | Jakarta EE namespace migration |
| rabbitmq-sender | Spring Boot 3.3.0 | Spring Boot 3.4.x | Minor updates |
| todo-web-api-use-oracle-db | Spring Boot 3.2.4 | Spring Boot 3.4.x | Minor updates |
| jakarta-ee/student-web-app | Java EE | Jakarta EE 10 | Namespace migration |

### Phase 3: Azure Service Migration

#### 3.1 Database Migration

**Oracle to Azure PostgreSQL (todo-web-api-use-oracle-db):**
- Replace Oracle-specific SQL (VARCHAR2, sequences, PL/SQL)
- Migrate to PostgreSQL syntax and data types
- Update JDBC driver from ojdbc to postgresql
- Migrate JPA entity mappings
- Test data type compatibility
- Performance tuning for PostgreSQL

**SQL Server with Managed Identity (mi-sql-public-demo):**
- Replace connection string with managed identity
- Update JDBC configuration for Azure AD authentication
- Implement DefaultAzureCredential pattern
- Remove hardcoded passwords from configuration

**PostgreSQL Enhancement (asset-manager):**
- Implement Azure Managed Identity authentication
- Configure Azure Database for PostgreSQL Flexible Server
- Enable high availability and backup features

#### 3.2 Storage Migration

**AWS S3 to Azure Blob Storage (asset-manager):**
- Replace AWS SDK with Azure Storage Blob SDK
- Migrate S3Client to BlobServiceClient
- Update bucket operations to container operations
- Migrate object key patterns to blob naming
- Update multipart upload logic
- Implement SAS tokens for secure access
- Add Azure Storage emulator support for local development

#### 3.3 Messaging Migration

**RabbitMQ to Azure Service Bus (asset-manager):**
- Replace Spring AMQP with Spring Cloud Azure Service Bus
- Migrate queue definitions to Service Bus queues/topics
- Update message producers and consumers
- Implement dead-letter queue patterns
- Configure retry policies
- Add connection string or managed identity auth

**RabbitMQ to Azure Service Bus (rabbitmq-sender):**
- Already completed in current state
- Validate implementation and test

#### 3.4 Authentication & Security

**Implement Managed Identity across all applications:**
- Replace password-based authentication
- Configure Azure Managed Identity (System or User-assigned)
- Update connection strings to use DefaultAzureCredential
- Remove secrets from application.properties
- Integrate with Azure Key Vault for sensitive configuration
- Implement proper RBAC roles

### Phase 4: Containerization

#### 4.1 Dockerfile Creation

For each application:
- Create multi-stage Dockerfiles for optimal image size
- Use appropriate base images (Eclipse Temurin, Microsoft OpenJDK)
- Implement proper layering for dependency caching
- Add health check endpoints
- Configure non-root user execution
- Optimize for Azure Container Registry

#### 4.2 Docker Compose for Local Development

- Create docker-compose.yml for local testing
- Include Azure service emulators (Azurite, Azure Service Bus emulator)
- Configure local PostgreSQL
- Set up development environment variables

### Phase 5: Infrastructure as Code

#### 5.1 Azure Bicep Templates

Create Bicep templates for:
- Resource Group
- Azure Container Apps Environment
- Container Apps (web and worker services)
- Azure Database for PostgreSQL Flexible Server
- Azure Storage Account with Blob containers
- Azure Service Bus namespace with queues/topics
- Azure Container Registry
- Azure Application Insights
- Virtual Network and private endpoints (optional)
- Azure Key Vault for secrets management

#### 5.2 Alternative: Terraform

If preferred, create equivalent Terraform configurations:
- azurerm provider configuration
- Resource modules for all Azure services
- Variable files for environment-specific configs
- Output values for service endpoints

### Phase 6: CI/CD Pipeline

#### 6.1 GitHub Actions Workflows

Create workflows for:
- **Build & Test**: Compile, unit tests, integration tests
- **Security Scanning**: CodeQL, dependency scanning, CVE checks
- **Container Build**: Build and push Docker images to ACR
- **Infrastructure Provisioning**: Deploy Bicep/Terraform templates
- **Application Deployment**: Deploy containers to Azure Container Apps
- **Environment Management**: Separate workflows for dev/staging/prod

#### 6.2 Azure DevOps Alternative

If using Azure DevOps:
- Create build pipelines
- Create release pipelines
- Configure service connections
- Set up approval gates

### Phase 7: Monitoring & Observability

#### 7.1 Application Insights Integration

- Add Azure Application Insights SDK
- Configure auto-instrumentation for Spring Boot apps
- Implement custom telemetry for business metrics
- Set up availability tests
- Configure alerts and notifications

#### 7.2 Health Endpoints

- Expose Spring Boot Actuator endpoints
- Configure liveness and readiness probes
- Implement custom health indicators
- Set up Azure Container Apps health checks

#### 7.3 Logging Strategy

- Centralize logs to Azure Log Analytics
- Implement structured logging (JSON format)
- Configure log retention policies
- Set up log queries and dashboards

### Phase 8: Testing & Validation

#### 8.1 Build Verification
- Compile all applications successfully
- Run unit tests
- Run integration tests
- Verify no build errors or warnings

#### 8.2 Security Validation
- Run CVE vulnerability scans
- Fix identified vulnerabilities
- Validate no new CVEs introduced
- Security code scanning with CodeQL

#### 8.3 Consistency Validation
- Verify behavior consistency pre/post migration
- Validate critical business flows
- Performance testing
- Load testing

#### 8.4 Completeness Validation
- Ensure all old technology references removed
- Verify all code paths migrated
- Check for leftover configuration
- Documentation completeness

### Phase 9: Deployment & Migration

#### 9.1 Deployment Strategy

**Blue-Green Deployment:**
- Deploy new Azure infrastructure (green)
- Run parallel with existing system (blue)
- Validate functionality in green environment
- Switch traffic to green
- Decommission blue after validation period

**Database Migration:**
- Backup existing databases
- Use Azure Database Migration Service for online migration
- Validate data integrity
- Update connection strings
- Switch applications to Azure databases

#### 9.2 Rollback Plan
- Maintain previous infrastructure during transition
- Document rollback procedures
- Test rollback scenarios
- Define rollback decision criteria

### Phase 10: Documentation & Training

#### 10.1 Documentation Updates

- Architecture diagrams for Azure deployment
- Deployment runbooks
- Troubleshooting guides
- Configuration management documentation
- Disaster recovery procedures
- Cost optimization guidelines

#### 10.2 Training Materials

- Developer onboarding guide
- Azure services overview
- Managed Identity usage
- Monitoring and alerting guide
- Incident response procedures

## Migration Task Breakdown by Application

### Asset Manager (Priority: HIGH)

**Complexity**: High - Multi-service application with AWS dependencies

**Tasks**:
1. Upgrade Java 8 → Java 21
2. Upgrade Spring Boot 2.7.18 → 3.4.x
3. Migrate AWS S3 → Azure Blob Storage
4. Migrate RabbitMQ → Azure Service Bus
5. Implement Managed Identity for PostgreSQL
6. Add Spring Boot Actuator health endpoints
7. Create Dockerfiles for web and worker modules
8. Create Azure infrastructure (Bicep/Terraform)
9. Set up CI/CD pipeline
10. Deploy to Azure Container Apps

**Estimated Effort**: 40-60 hours

### Todo Web API with Oracle Database (Priority: HIGH)

**Complexity**: High - Database migration with Oracle-specific features

**Tasks**:
1. Migrate Oracle Database → Azure Database for PostgreSQL
2. Replace Oracle-specific SQL and data types
3. Update JDBC driver and JPA configuration
4. Implement Managed Identity authentication
5. Add health endpoints
6. Create Dockerfile
7. Create Azure infrastructure
8. Set up CI/CD pipeline
9. Deploy to Azure Container Apps

**Estimated Effort**: 30-40 hours

### MI SQL Public Demo (Priority: MEDIUM)

**Complexity**: Low - Simple authentication migration

**Tasks**:
1. Replace password authentication → Managed Identity
2. Update connection configuration
3. Add health endpoints
4. Create Dockerfile
5. Deploy to Azure App Service or Container Apps

**Estimated Effort**: 8-12 hours

### Jakarta EE Student Web App (Priority: MEDIUM)

**Complexity**: High - Legacy build system and framework migration

**Tasks**:
1. Migrate Ant build → Maven
2. Migrate Java EE → Jakarta EE 10
3. Update Servlet APIs and namespaces
4. Modernize database connection handling
5. Add health endpoints
6. Create Dockerfile
7. Deploy to Azure App Service or Container Apps

**Estimated Effort**: 40-50 hours

### RabbitMQ Sender (Priority: LOW)

**Complexity**: Low - Already mostly migrated

**Tasks**:
1. Validate current Azure Service Bus integration
2. Add comprehensive tests
3. Create Dockerfile
4. Create deployment pipeline

**Estimated Effort**: 4-8 hours

## Cost Optimization Recommendations

### Right-Sizing Resources
- Start with smaller SKUs and scale up as needed
- Use Azure Container Apps consumption plan for variable workloads
- Enable autoscaling based on metrics

### Reserved Instances
- Consider reserved instances for production databases
- Use reserved capacity for predictable workloads

### Development/Test Pricing
- Use dev/test pricing for non-production environments
- Shut down non-production resources outside business hours

### Storage Optimization
- Use appropriate storage tiers (hot/cool/archive)
- Implement lifecycle management policies
- Enable compression where applicable

### Monitoring Costs
- Set appropriate log retention periods
- Use sampling for high-volume telemetry
- Configure alert budgets

## Risk Assessment & Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Data loss during migration | Critical | Low | Comprehensive backup strategy, parallel run |
| Downtime during cutover | High | Medium | Blue-green deployment, maintenance window |
| Performance degradation | High | Medium | Load testing, performance baseline |
| Cost overruns | Medium | Medium | Cost monitoring, budget alerts |
| Security vulnerabilities | Critical | Low | Security scanning, penetration testing |
| Compatibility issues | High | Medium | Thorough testing, staged rollout |
| Knowledge gaps | Medium | Medium | Training, documentation, expert consultation |

## Success Criteria

### Technical Success Metrics
- ✅ All applications successfully deployed to Azure
- ✅ Zero critical or high security vulnerabilities
- ✅ All unit and integration tests passing
- ✅ Performance meets or exceeds baseline
- ✅ 99.9% availability target met
- ✅ All Azure best practices implemented

### Business Success Metrics
- ✅ Migration completed within estimated timeline
- ✅ Zero data loss or corruption
- ✅ Minimal downtime during cutover
- ✅ Cost within projected budget
- ✅ Team trained on Azure platform
- ✅ Documentation complete and accessible

## Timeline & Milestones

### Month 1: Assessment & Planning
- Week 1-2: Complete application assessment
- Week 3-4: Finalize architecture and infrastructure design

### Month 2: Development Environment Setup
- Week 1: Set up Azure subscriptions and resources
- Week 2: Create development environments
- Week 3: Set up CI/CD pipelines
- Week 4: Team training on Azure services

### Month 3-4: Application Modernization
- Java and framework upgrades
- Azure service migrations
- Containerization
- Testing and validation

### Month 5: Deployment & Validation
- Staging environment deployment
- Load and performance testing
- Security validation
- Production deployment preparation

### Month 6: Production Migration & Support
- Production deployment
- Monitoring and optimization
- Documentation finalization
- Post-migration support

## Tools & Technologies

### Migration Tools
- **GitHub Copilot App Modernization**: AI-assisted migration
- **Azure Migrate**: Assessment and migration planning
- **Azure Database Migration Service**: Database migration
- **AppCAT (Azure Platform Compatibility Toolkit)**: Compatibility assessment

### Development Tools
- **VS Code** or **IntelliJ IDEA**: IDE with Azure extensions
- **Maven/Gradle**: Build tools
- **Docker**: Containerization
- **Git**: Version control

### Azure Services
- **Azure Container Apps**: Serverless container hosting
- **Azure Database for PostgreSQL**: Managed database
- **Azure Blob Storage**: Object storage
- **Azure Service Bus**: Enterprise messaging
- **Azure Container Registry**: Container image registry
- **Azure Key Vault**: Secrets management
- **Azure Application Insights**: Monitoring and diagnostics
- **Azure DevOps** or **GitHub Actions**: CI/CD

### Infrastructure as Code
- **Azure Bicep**: Native Azure IaC
- **Terraform**: Alternative IaC option

## Next Steps

1. **Immediate Actions**:
   - Review and approve this modernization plan
   - Set up Azure subscription and resource groups
   - Configure development environments
   - Begin with MI SQL Public Demo (quick win)

2. **Week 1 Actions**:
   - Start Asset Manager assessment
   - Begin Java/Spring Boot upgrades
   - Set up CI/CD pipeline foundations

3. **Week 2 Actions**:
   - Start database migration planning
   - Begin containerization work
   - Create infrastructure templates

4. **Continuous Actions**:
   - Track progress in GitHub Issues/Projects
   - Regular team sync meetings
   - Document learnings and best practices
   - Monitor costs and optimize

## References

- [Azure Migration Documentation](https://docs.microsoft.com/azure/migrate/)
- [Spring Cloud Azure](https://spring.io/projects/spring-cloud-azure)
- [Azure Container Apps](https://docs.microsoft.com/azure/container-apps/)
- [Azure Database for PostgreSQL](https://docs.microsoft.com/azure/postgresql/)
- [GitHub Copilot App Modernization](https://code.visualstudio.com/docs/copilot/overview)
- [Azure Architecture Center](https://docs.microsoft.com/azure/architecture/)

## Appendix A: Azure Service Mapping

| Current Technology | Azure Equivalent | Migration Complexity |
|-------------------|------------------|---------------------|
| AWS S3 | Azure Blob Storage | Medium |
| RabbitMQ | Azure Service Bus | Medium |
| Oracle Database | Azure Database for PostgreSQL | High |
| SQL Server (on-prem) | Azure SQL Database | Low |
| Java EE | Jakarta EE on Azure App Service | Medium |
| Password Authentication | Azure Managed Identity | Low |
| Local File System | Azure Blob Storage | Low |

## Appendix B: Azure Deployment Architectures

### Asset Manager Architecture (Target)
```
┌─────────────────────────────────────────────────────────────┐
│                      Azure Subscription                       │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────┐ │
│  │          Azure Container Apps Environment               │ │
│  │  ┌──────────────┐         ┌──────────────┐            │ │
│  │  │   Web App    │────────▶│  Worker App  │            │ │
│  │  │ (Container)  │         │ (Container)  │            │ │
│  │  └───────┬──────┘         └──────┬───────┘            │ │
│  └──────────┼──────────────────────┼─────────────────────┘ │
│             │                       │                        │
│             ▼                       ▼                        │
│  ┌──────────────────┐    ┌──────────────────┐             │
│  │  Azure Blob      │    │  Azure Service   │             │
│  │  Storage         │    │  Bus             │             │
│  └──────────────────┘    └──────────────────┘             │
│             │                                               │
│             ▼                                               │
│  ┌──────────────────────────────────────┐                 │
│  │  Azure Database for PostgreSQL       │                 │
│  │  Flexible Server                      │                 │
│  └──────────────────────────────────────┘                 │
│                                                             │
│  ┌──────────────────────────────────────┐                 │
│  │  Azure Application Insights           │                 │
│  └──────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

All services connected via Azure Managed Identity for secure authentication.

---

**Document Version**: 1.0  
**Last Updated**: January 16, 2026  
**Status**: Initial Plan  
**Owner**: Migration Team
