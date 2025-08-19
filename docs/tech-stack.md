# Technology Stack

## Overview

The Azure Data Analytics Platform leverages modern cloud-native services for scalability, security, and cost optimization. Each technology choice is driven by specific technical requirements and integration capabilities.

## Data Storage & Processing

### Azure Data Lake Storage Gen2
**Purpose**: Unified data lake for all data layers (Bronze/Silver/Gold)

**Key Features**:
- Hierarchical namespace for better performance
- Lifecycle management for cost optimization  
- Integration with Azure services
- Granular access controls with RBAC

**Configuration**:
```json
{
  "storageAccountType": "Standard_LRS", 
  "accessTier": "Hot",
  "hierarchicalNamespace": true,
  "encryption": "Microsoft.Storage"
}
```

### Azure Databricks
**Purpose**: Unified analytics platform for ETL and data science workloads

**Key Features**:
- Apache Spark-based distributed computing
- Delta Lake for ACID transactions  
- Auto-scaling clusters
- Collaborative notebooks
- MLflow for ML lifecycle management

**Cluster Configuration**:
```json
{
  "clusterName": "etl-cluster",
  "sparkVersion": "13.3.x-scala2.12",
  "nodeTypeId": "Standard_DS3_v2",
  "minWorkers": 2,
  "maxWorkers": 8,
  "autoTerminationMinutes": 30
}
```

### Delta Lake
**Purpose**: Storage layer providing ACID transactions and time travel

**Benefits**:
- Schema enforcement and evolution
- Unified streaming and batch processing
- Data versioning and rollback capabilities
- Optimized file formats (Parquet + transaction log)

## Compute & Orchestration

### Azure Data Factory
**Purpose**: Data integration and orchestration service

**Use Cases**:
- Trigger Databricks notebooks on schedule
- Copy data between different sources
- Monitor pipeline execution and failures
- Parameterized workflows for different environments

**Pipeline Example**:
```json
{
  "name": "bronze-to-silver-pipeline",
  "activities": [
    {
      "name": "run-bronze-ingestion", 
      "type": "DatabricksNotebook",
      "notebookPath": "/bronze-ingestion/ingest-sales-data"
    }
  ],
  "triggers": [
    {
      "name": "daily-trigger",
      "type": "ScheduleTrigger", 
      "recurrence": "Daily 2:00 AM UTC"
    }
  ]
}
```

### Azure App Service
**Purpose**: Host .NET Web API with auto-scaling capabilities

**Configuration**:
- **Runtime**: .NET 8.0
- **OS**: Linux (cost-optimized)
- **Tier**: Standard S1 (dev), Premium P1V2 (prod)
- **Auto-scaling**: CPU percentage > 70%

## Development & Programming Languages

### C# (.NET 8)
**Purpose**: API layer development with modern language features

**Architecture**: Clean Architecture with dependency injection
- **Controllers**: HTTP request handling
- **Services**: Business logic implementation  
- **Repositories**: Data access abstraction
- **Models**: Data transfer objects

**Key Libraries**:
```xml
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" />
<PackageReference Include="Microsoft.Identity.Web" />
<PackageReference Include="Azure.Identity" />
<PackageReference Include="Microsoft.EntityFrameworkCore" />
```

### Python (PySpark)
**Purpose**: ETL development in Databricks notebooks

**Key Libraries**:
```python
from pyspark.sql import SparkSession
from delta.tables import DeltaTable
from pyspark.sql.functions import col, when, regexp_replace
import great_expectations  # Data quality validation
```

**Code Standards**:
- Type hints for all functions
- Docstring documentation
- Unit tests with pytest
- Code formatting with black

### PowerShell
**Purpose**: Infrastructure deployment and Power BI automation

**Use Cases**:
- Azure resource deployment scripts
- Power BI dataset refresh automation
- Environment configuration management
- CI/CD pipeline tasks

## Authentication & Security

### Azure Active Directory (Azure AD)
**Purpose**: Identity and access management across all services

**Features**:
- Single sign-on (SSO) integration
- Multi-factor authentication (MFA)
- Conditional access policies
- Application registration for API access

**API Scopes**:
```json
{
  "api://dataplatform/read": "Read-only access to data",
  "api://dataplatform/write": "Write access to data", 
  "api://dataplatform/admin": "Administrative operations"
}
```

### Managed Identity
**Purpose**: Service-to-service authentication without storing secrets

**Benefits**:
- Eliminates need for connection strings in code
- Automatic credential rotation
- Least privilege access principles
- Integration with Azure Key Vault

### Azure Key Vault
**Purpose**: Secure storage of secrets, keys, and certificates

**Stored Secrets**:
- Database connection strings
- API keys for external services
- Storage account access keys
- Service principal credentials

## Business Intelligence & Visualization

### Power BI Premium
**Purpose**: Enterprise-scale business intelligence platform

**Capabilities**:
- Large dataset support (>10GB)
- Dedicated compute capacity
- Advanced analytics with R/Python
- Automated refresh scheduling
- Row-level security (RLS)

**Data Connection Methods**:
```json
{
  "directQuery": {
    "description": "Real-time data access",
    "latency": "< 1 second",
    "use_case": "Operational dashboards"
  },
  "import": {
    "description": "Cached data for performance", 
    "refresh": "Scheduled hourly",
    "use_case": "Historical analysis"
  }
}
```

### Power BI REST API
**Purpose**: Programmatic management of datasets and reports

**Use Cases**:
- Automated dataset refresh
- Report deployment across environments
- User access management
- Usage analytics collection

## DevOps & CI/CD

### Azure DevOps
**Purpose**: Source control, CI/CD pipelines, and project management

**Components**:
- **Azure Repos**: Git repositories with branch policies
- **Azure Pipelines**: CI/CD automation
- **Azure Boards**: Work item tracking
- **Azure Artifacts**: Package management

### Infrastructure as Code (Bicep)
**Purpose**: Declarative Azure resource management

**Benefits**:
- Version-controlled infrastructure
- Consistent deployments across environments
- Parameter-driven configurations
- Dependency management

**Template Structure**:
```bicep
@description('Environment name (dev/staging/prod)')
param environment string = 'dev'

@description('Resource group location')
param location string = resourceGroup().location

// Resource definitions with parameters
resource storageAccount 'Microsoft.Storage/storageAccounts@2021-09-01' = {
  name: 'stdataplatform${environment}${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    // Configuration based on environment
  }
}
```

## Monitoring & Observability

### Azure Monitor
**Purpose**: Comprehensive monitoring solution for applications and infrastructure

**Components**:
- **Metrics**: Performance counters and custom metrics
- **Logs**: Application and system logs via Log Analytics
- **Alerts**: Proactive notifications based on thresholds
- **Dashboards**: Unified view of system health

### Application Insights
**Purpose**: Application performance monitoring for .NET API

**Tracked Metrics**:
- Request/response times
- Dependency call durations
- Exception rates and details
- Custom business metrics

**Configuration**:
```csharp
services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = configuration["ApplicationInsights:ConnectionString"];
});
```

### Databricks Monitoring
**Purpose**: ETL pipeline monitoring and data lineage

**Features**:
- Job execution metrics and logs
- Cluster utilization monitoring
- Data lineage visualization
- Custom metric collection

## Cost Optimization Strategies

### Compute Optimization
- **Databricks**: Job clusters vs. all-purpose clusters (-60% cost)
- **Azure App Service**: Linux hosting vs. Windows (-20% cost)
- **Storage**: Hot/Cool/Archive tiers based on access patterns

### Auto-scaling Configuration
```json
{
  "databricks": {
    "minWorkers": 2,
    "maxWorkers": 8, 
    "autoTermination": "30 minutes"
  },
  "appService": {
    "scaleOutRules": "CPU > 70% for 5 minutes",
    "scaleInRules": "CPU < 30% for 15 minutes"
  }
}
```

### Reserved Instances
- **Production Environment**: 1-year reserved instances (-30% cost)
- **Development**: Spot instances where applicable (-90% cost)

## Development Tools & IDEs

### Recommended Development Environment
- **Visual Studio Code**: Primary IDE with extensions
- **Azure CLI**: Command-line Azure management
- **Git**: Version control with branch protection rules
- **Postman**: API testing and documentation
- **Power BI Desktop**: Report development and testing

### VS Code Extensions
```json
{
  "recommendations": [
    "ms-vscode.azure-account",
    "ms-azuretools.vscode-bicep", 
    "ms-python.python",
    "ms-dotnettools.csharp",
    "ms-vscode.powershell"
  ]
}
```

## Data Format Standards

### File Formats
- **Bronze Layer**: Original format (CSV, JSON, Parquet)
- **Silver/Gold**: Delta format for ACID compliance
- **Export**: Parquet for external systems

### Naming Conventions
```
Storage Accounts: stdataplatform{env}{random}
Resource Groups: rg-dataplatform-{env}
Key Vaults: kv-dataplatform-{env}
Databricks Workspaces: dbw-dataplatform-{env}
```

### Schema Evolution Strategy
- **Backward Compatible**: New optional columns only
- **Version Control**: Schema changes tracked in Git
- **Migration Scripts**: Automated data transformation for breaking changes