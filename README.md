# Azure Data Analytics Platform

A comprehensive multi-agent data platform built on Azure that ingests, transforms, and serves data through ETL pipelines, Power BI dashboards, and REST APIs.

## 🏗️ Architecture

This project implements a **Bronze → Silver → Gold** data architecture using Azure services:

- **Bronze Layer**: Raw data ingestion from multiple sources
- **Silver Layer**: Cleaned and standardized data  
- **Gold Layer**: Analytics-ready star schema for reporting

See [docs/architecture.md](docs/architecture.md) for detailed architecture overview.

## 📊 Data Schema

The platform handles three main data domains:
- **Sales Data** (ERP/CRM) - Orders, Subscriptions, Revenue
- **Customer Data** (CRM) - Profiles, Segments, Support  
- **IoT Device Data** (Telemetry) - Usage, Health, Errors

See [docs/schema.md](docs/schema.md) for complete data schema definitions.

## 🛠️ Tech Stack

- **Data Lake**: Azure Data Lake Storage Gen2
- **Compute & ETL**: Azure Databricks (PySpark + SQL)
- **Orchestration**: Azure Data Factory
- **Visualization**: Power BI
- **API Layer**: C# (.NET 8) Web API + Azure App Service
- **Authentication**: Azure AD + Managed Identity

See [docs/tech-stack.md](docs/tech-stack.md) for detailed technology choices.

## 🤖 Multi-Agent Development

This project uses a multi-agent approach with specialized agents:

1. **Infrastructure Agent** - Azure resources, DevOps pipelines, RBAC
2. **Databricks Agent** - ETL notebooks, data transformations  
3. **.NET API Agent** - REST API, authentication, data access
4. **Power BI Agent** - Dashboards, reports, deployment

Each agent has its own context file in the respective directories.

## 📁 Project Structure

```
azure-data-platform/
├── .azure-devops/          # CI/CD pipelines and templates
├── infrastructure/         # Bicep templates and deployment scripts
├── src/
│   ├── databricks/         # ETL notebooks and jobs
│   ├── api/               # .NET Web API project
│   └── powerbi/           # Power BI templates and scripts
├── data/                  # Sample data and schema definitions
├── docs/                  # Documentation
└── tests/                 # End-to-end and performance tests
```

## 🚀 Quick Start

1. **Prerequisites**: Azure subscription, Azure CLI, Power BI Pro license
2. **Infrastructure**: Deploy Azure resources using Bicep templates
3. **Data Pipeline**: Set up Databricks workspace and run ETL notebooks
4. **API**: Deploy .NET API to Azure App Service
5. **Dashboards**: Import Power BI templates and configure data sources

## 📖 Documentation

- [Architecture Overview](docs/architecture.md)
- [Data Schema Reference](docs/schema.md)
- [Technology Stack](docs/tech-stack.md)
- [Deployment Guide](docs/deployment-guide.md)
- [API Documentation](docs/api-documentation.md)
- [Troubleshooting](docs/troubleshooting.md)

## 🎯 Key Features

- **Real-time IoT ingestion** from Azure IoT Hub
- **Automated ETL pipelines** with data quality checks
- **Interactive Power BI dashboards** for Customer 360, Device Health, and Revenue Analytics
- **Secure REST APIs** with Azure AD authentication
- **Cost-optimized** compute with auto-scaling Databricks clusters
- **Multi-environment** support (dev/staging/prod)

## 📈 Success Metrics

- 95% successful ingestion jobs
- <5% data quality errors  
- <1s response time for API queries
- Power BI adoption: 50+ daily active users

## 🤝 Contributing

Each component has its own context file for specialized development:
- Infrastructure: `infrastructure/context.md`
- Databricks: `src/databricks/context.md`  
- API: `src/api/context.md`
- Power BI: `src/powerbi/context.md`

## 📄 License

This project is licensed under the MIT License.