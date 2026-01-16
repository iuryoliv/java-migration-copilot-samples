# Java Migration Copilot Samples

This repository contains sample Java applications demonstrating migration to Azure using GitHub Copilot app modernization. Each sample showcases different migration scenarios and best practices for moving Java applications to Azure cloud services.

## 📋 Azure Modernization Documentation

**NEW**: Comprehensive guides for modernizing and deploying Java applications to Azure:

- **[Azure Modernization Plan](AZURE_MODERNIZATION_PLAN.md)** - Complete strategy and roadmap for migrating all applications to Azure
- **[Azure Deployment Guide](AZURE_DEPLOYMENT_GUIDE.md)** - Step-by-step infrastructure provisioning and deployment instructions
- **[CI/CD Pipeline Guide](CI_CD_PIPELINE_GUIDE.md)** - GitHub Actions workflow templates for automated deployments
- **[Azure Service Mapping Reference](AZURE_SERVICE_MAPPING.md)** - Quick reference for technology-to-Azure service mappings

These guides provide a complete blueprint for:
- ✅ Upgrading Java versions (8/11 → 17/21)
- ✅ Upgrading Spring Boot (2.x → 3.x) and frameworks
- ✅ Migrating from AWS/on-premises to Azure services
- ✅ Implementing Azure Managed Identity for secure authentication
- ✅ Containerizing applications with Docker
- ✅ Deploying to Azure Container Apps
- ✅ Setting up CI/CD pipelines with GitHub Actions
- ✅ Monitoring and observability with Application Insights

## 📦 Sample Applications

### 🚀 [Asset Manager](asset-manager) (Complete Workshop)
**Complexity**: High | **Priority**: High

A complete end-to-end workshop demonstrating migration of a multi-service application to Azure.

**Current Stack**: Java 8, Spring Boot 2.7.18, AWS S3, RabbitMQ, PostgreSQL  
**Target Stack**: Java 21, Spring Boot 3.4.x, Azure Blob Storage, Azure Service Bus, Azure Database for PostgreSQL

**Demonstrates**:
- Multi-module Maven project migration
- Java and Spring Boot version upgrades
- AWS to Azure service migration (S3 → Blob Storage, RabbitMQ → Service Bus)
- Managed Identity implementation
- Containerization with Docker
- Deployment to Azure Container Apps
- Health checks with Spring Boot Actuator

**Estimated Migration Time**: 40-60 hours

---

### 🗄️ [Todo Web API with Oracle Database](todo-web-api-use-oracle-db)
**Complexity**: High | **Priority**: High

A REST API application demonstrating migration from Oracle Database to Azure Database for PostgreSQL.

**Current Stack**: Java 17, Spring Boot 3.2.4, Oracle Database  
**Target Stack**: Java 21, Spring Boot 3.4.x, Azure Database for PostgreSQL

**Demonstrates**:
- Oracle to PostgreSQL database migration
- Oracle-specific SQL feature conversion (VARCHAR2, SEQUENCE, PL/SQL)
- JPA entity mapping updates
- JDBC driver migration

**Estimated Migration Time**: 30-40 hours

---

### 🔐 [MI SQL Public Demo](mi-sql-public-demo) (Quick Start)
**Complexity**: Low | **Priority**: Medium

Getting started sample showing migration from password-based authentication to Azure Managed Identity.

**Current Stack**: Java 17, SQL Server with password authentication  
**Target Stack**: Java 17/21, Azure SQL Database with Managed Identity

**Demonstrates**:
- Simplest Managed Identity implementation
- Secure database connections without passwords
- Azure AD authentication
- Quick win for security improvements

**Estimated Migration Time**: 8-12 hours

---

### 📨 [RabbitMQ Sender](rabbitmq-sender) (Custom Formula Example)
**Complexity**: Low | **Priority**: Low

Sample application demonstrating custom migration formulas and patterns.

**Current Stack**: Java 17, Spring Boot 3.3.0, RabbitMQ (already migrated to Azure Service Bus)  
**Target Stack**: Validation and testing needed

**Demonstrates**:
- Creating custom migration formulas
- Azure Service Bus integration
- Spring Cloud Azure messaging

**Estimated Migration Time**: 4-8 hours

---

### 🎓 [Student Web App - Jakarta EE](jakarta-ee/student-web-app)
**Complexity**: High | **Priority**: Medium

Legacy Java EE web application demonstrating framework and build system modernization.

**Current Stack**: Java 8, Java EE, Ant build, Open Liberty, MyBatis  
**Target Stack**: Java 17/21, Jakarta EE 10, Maven build, Azure App Service

**Demonstrates**:
- Ant to Maven migration
- Java EE to Jakarta EE namespace migration
- Servlet API updates
- Legacy application modernization
- Hybrid architecture (Servlets + Spring MVC)

**Estimated Migration Time**: 40-50 hours

## 🎯 Migration Priorities

Based on complexity and business value:

1. **High Priority**
   - Asset Manager (comprehensive showcase)
   - Todo Web API (database migration patterns)

2. **Medium Priority**
   - MI SQL Public Demo (quick security win)
   - Student Web App (legacy modernization)

3. **Low Priority**
   - RabbitMQ Sender (already mostly migrated)

## 🛠️ Prerequisites

### For Local Development
- **Java JDK**: Versions 8, 11, 17, and 21 (for different projects)
- **Maven**: 3.8.0 or later
- **Docker**: 20.10 or later
- **Git**: Latest version

### For Azure Deployment
- **Azure CLI**: 2.50.0 or later
- **Azure Subscription**: With appropriate permissions
- **GitHub Copilot**: Pro, Business, or Enterprise plan
- **VS Code** or **IntelliJ IDEA**: With GitHub Copilot and App Modernization extensions

## 🚀 Quick Start

### 1. Clone the Repository
```bash
git clone https://github.com/Azure-Samples/java-migration-copilot-samples.git
cd java-migration-copilot-samples
```

### 2. Review Documentation
Start with the [Azure Modernization Plan](AZURE_MODERNIZATION_PLAN.md) to understand the overall strategy.

### 3. Choose Your Path

**Option A: Complete Workshop** (Recommended for learning)
```bash
cd asset-manager
# Follow the detailed README and workshop guide
```

**Option B: Quick Win** (Fastest results)
```bash
cd mi-sql-public-demo
# Migrate to Managed Identity in < 1 day
```

**Option C: Database Migration** (Common scenario)
```bash
cd todo-web-api-use-oracle-db
# Learn Oracle to PostgreSQL migration patterns
```

### 4. Deploy to Azure
Follow the [Azure Deployment Guide](AZURE_DEPLOYMENT_GUIDE.md) for step-by-step instructions.

## 📚 Documentation Structure

```
├── AZURE_MODERNIZATION_PLAN.md    # Complete migration strategy
├── AZURE_DEPLOYMENT_GUIDE.md       # Infrastructure & deployment
├── CI_CD_PIPELINE_GUIDE.md         # GitHub Actions workflows
├── AZURE_SERVICE_MAPPING.md        # Technology mapping reference
├── asset-manager/
│   └── README.md                   # Complete workshop guide
├── todo-web-api-use-oracle-db/
│   └── README.md                   # Oracle migration guide
├── mi-sql-public-demo/
│   └── README.md                   # Managed Identity guide
├── rabbitmq-sender/
│   └── README.md                   # Custom formula guide
└── jakarta-ee/student-web-app/
    └── README.md                   # Jakarta EE migration guide
```

## 🌿 Branches

- **`main`**: Source projects in their original state (before migration)
- **`expected`**: Expected state after migration (reference implementation)
- **`workshop/*`**: Workshop-specific branches with intermediate states

## 🎓 Learning Path

### Beginner
1. Start with **MI SQL Public Demo** (Managed Identity basics)
2. Review **Azure Service Mapping Reference**
3. Try **RabbitMQ Sender** (messaging patterns)

### Intermediate
1. Complete **Asset Manager Workshop** (comprehensive migration)
2. Implement CI/CD pipelines from templates
3. Deploy to Azure Container Apps

### Advanced
1. Tackle **Todo Web API** (complex database migration)
2. Modernize **Student Web App** (legacy application)
3. Customize for your own applications

## 💰 Cost Estimates

Rough monthly costs for running migrated applications on Azure:

| Service | Development | Production |
|---------|------------|------------|
| Azure Container Apps | $50-100 | $100-200 |
| Azure Database for PostgreSQL | $30-60 | $50-150 |
| Azure Blob Storage | $5-10 | $10-20 |
| Azure Service Bus | $5-10 | $10-50 |
| Application Insights | $20-40 | $50-100 |
| **Total Estimated** | **$110-220** | **$220-520** |

*Use [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) for accurate estimates.*

## 🤝 Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## 📄 License

This project is licensed under the MIT License - see [LICENSE.md](LICENSE.md) for details.

## 🆘 Support & Resources

### Documentation
- [GitHub Copilot App Modernization Docs](https://code.visualstudio.com/docs/copilot/overview)
- [Azure Java Developer Center](https://docs.microsoft.com/azure/developer/java/)
- [Spring Cloud Azure](https://spring.io/projects/spring-cloud-azure)

### Tools
- [GitHub Copilot App Modernization Extension](https://marketplace.visualstudio.com/items?itemName=vscjava.migrate-java-to-azure)
- [Azure Migrate](https://azure.microsoft.com/services/azure-migrate/)
- [Azure TCO Calculator](https://azure.microsoft.com/pricing/tco/)

### Community
- [Microsoft Q&A - Java on Azure](https://docs.microsoft.com/answers/topics/azure-java-sdk.html)
- [GitHub Discussions](https://github.com/Azure-Samples/java-migration-copilot-samples/discussions)

## 📝 Changelog

See [CHANGELOG.md](CHANGELOG.md) for a list of changes and updates.

---

**Getting Started?** Begin with the [Azure Modernization Plan](AZURE_MODERNIZATION_PLAN.md) 📖

**Questions?** Check out the [GitHub Discussions](https://github.com/Azure-Samples/java-migration-copilot-samples/discussions) 💬

**Ready to Deploy?** Follow the [Azure Deployment Guide](AZURE_DEPLOYMENT_GUIDE.md) 🚀
