# Deployment Guide

Quick deployment guide with links to detailed agent contexts.

## Prerequisites
- Azure subscription with sufficient quotas
- Azure CLI installed and authenticated
- Power BI Pro/Premium license
- Git repository access

## Deployment Order

### 1. Infrastructure Deployment
Deploy Azure resources using the Infrastructure Agent.

See [infrastructure/context.md](../infrastructure/context.md) for detailed implementation steps.

### 2. Databricks ETL Setup  
Configure data pipelines using the Databricks Agent.

See [src/databricks/context.md](../src/databricks/context.md) for notebook development and job configuration.

### 3. API Deployment
Deploy .NET Web API using the API Agent.

See [src/api/context.md](../src/api/context.md) for API development and Azure AD integration.

### 4. Power BI Dashboard Creation
Create and deploy dashboards using the Power BI Agent.

See [src/powerbi/context.md](../src/powerbi/context.md) for dashboard development and automation.

## Environment Configuration

Each environment (dev/staging/prod) has specific configurations documented in the respective agent context files.