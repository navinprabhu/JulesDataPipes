# Architecture Overview

## System Architecture

The Azure Data Analytics Platform follows a modern data architecture pattern with clear separation of concerns across four specialized components.

```
[Data Sources]           [Ingestion]              [Processing]            [Serving]
     ↓                       ↓                        ↓                     ↓
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌───────────────┐
│ CRM/ERP/IoT │ → │ Azure Blob      │ → │ Databricks ETL  │ → │ Power BI      │
│ Systems     │   │ IoT Hub         │   │ Bronze→Silver   │   │ Dashboards    │
│             │   │ Data Lake Gen2  │   │ →Gold Pipeline  │   │               │
└─────────────┘     └─────────────────┘     └─────────────────┘     ├───────────────┤
                                                    ↓                │ C# .NET API   │
                                            ┌─────────────────┐     │ Azure App     │
                                            │ Delta Lake      │ ← │ Service       │
                                            │ Synapse (opt)   │     └───────────────┘
                                            └─────────────────┘
```

## Data Flow Architecture

### 1. Data Ingestion Layer
- **Batch Ingestion**: CSV/JSON files uploaded to Azure Blob Storage
- **Streaming Ingestion**: Real-time IoT events via Azure IoT Hub
- **Orchestration**: Azure Data Factory triggers and schedules data pipelines

### 2. Data Processing Layer (Databricks)
- **Bronze Layer**: Raw data stored as-is with minimal schema enforcement
- **Silver Layer**: Cleaned, deduplicated, and standardized data
- **Gold Layer**: Business-ready dimensional model (star schema)

### 3. Data Serving Layer
- **Power BI**: Interactive dashboards connected to Gold layer
- **REST API**: Secure endpoints for external applications
- **Direct Query**: Real-time access to latest data

## Multi-Agent Architecture

### Agent Responsibilities

#### 1. Infrastructure Agent 🏗️
- **Scope**: Azure resources, CI/CD pipelines, security
- **Deliverables**:
  - Bicep templates for all Azure services
  - Azure DevOps pipelines with approval gates
  - RBAC configuration and managed identities
  - Cost optimization and monitoring setup

#### 2. Databricks Agent 🔄
- **Scope**: Data engineering, ETL pipelines, data quality
- **Deliverables**:
  - PySpark notebooks for each transformation layer
  - Databricks job definitions and scheduling
  - Data quality checks and schema validation
  - Comprehensive test suites

#### 3. .NET API Agent 🌐
- **Scope**: Backend services, authentication, data access
- **Deliverables**:
  - .NET 8 Web API with clean architecture
  - Azure AD authentication integration
  - Data access layer for Gold tables
  - API documentation and testing

#### 4. Power BI Agent 📊
- **Scope**: Business intelligence, visualization, reporting
- **Deliverables**:
  - Power BI templates (.pbit files)
  - Automated deployment scripts
  - Row-level security implementation
  - Performance optimization

## Security Architecture

### Authentication & Authorization
- **Azure Active Directory**: Single sign-on across all components
- **Managed Identity**: Service-to-service authentication without secrets
- **API Scopes**: Granular permissions (read/write/admin)
- **Row-Level Security**: Data access based on user roles

### Data Protection
- **Encryption at Rest**: All storage encrypted with Azure-managed keys
- **Encryption in Transit**: TLS 1.2+ for all communications
- **Network Security**: Private endpoints and VNet integration
- **Secrets Management**: Azure Key Vault for all sensitive configuration

## Scalability & Performance

### Compute Scaling
- **Databricks Clusters**: Auto-scaling based on workload
- **Azure App Service**: Horizontal scaling for API layer
- **Power BI Premium**: Dedicated capacity for large datasets

### Storage Optimization
- **Delta Lake**: Optimized file formats with Z-ordering
- **Partitioning**: Time-based partitioning for efficient queries
- **Lifecycle Management**: Automated archival of old data

### Cost Optimization
- **Job Clusters**: Terminate after ETL completion
- **Reserved Instances**: Cost savings for production workloads
- **Data Tiering**: Hot/cool/archive storage based on access patterns

## Monitoring & Observability

### Application Monitoring
- **Azure Monitor**: Centralized logging and metrics
- **Application Insights**: API performance and error tracking
- **Databricks Monitoring**: Job execution metrics and lineage

### Data Quality Monitoring
- **Schema Drift Detection**: Automated alerts for data changes
- **Data Quality Metrics**: Completeness, accuracy, consistency
- **Pipeline Health**: Success rates and execution times

## Disaster Recovery

### Backup Strategy
- **Data**: Geo-redundant storage with automated backups
- **Code**: Git-based version control with branch protection
- **Configuration**: Infrastructure as Code for reproducibility

### Recovery Procedures
- **RTO**: 4 hours for production workloads
- **RPO**: 1 hour maximum data loss
- **Testing**: Quarterly DR drills and documentation updates

## Environment Strategy

### Development
- **Purpose**: Agent development and initial testing
- **Resources**: Basic SKUs, single region
- **Data**: Synthetic data only, no PII

### Staging  
- **Purpose**: Integration testing and user acceptance
- **Resources**: Standard SKUs, production-like setup
- **Data**: Masked production data

### Production
- **Purpose**: Live business operations
- **Resources**: Premium SKUs with high availability
- **Data**: Full production dataset with security controls