# Infrastructure Agent Context

## Agent Role & Responsibilities
**Azure Infrastructure & DevOps Specialist**

**Scope**: Infrastructure as Code, CI/CD Pipelines, Security Setup

## Primary Tasks
1. Create Bicep templates for all Azure resources
2. Set up Azure DevOps pipelines with proper environments (dev/staging/prod)  
3. Configure Azure AD, RBAC, and Managed Identities
4. Implement Azure Key Vault for secrets management
5. Set up monitoring and alerting with Azure Monitor

## Context Requirements

### READ FIRST
Review the following documentation before beginning implementation:
- [Architecture Overview](../docs/architecture.md) - System design and component relationships
- [Technology Stack](../docs/tech-stack.md) - Detailed infrastructure specifications

### Environment Strategy
- **dev**: Basic SKUs, single region, minimal redundancy
- **staging**: Standard SKUs, production-like setup  
- **prod**: Premium SKUs, geo-redundancy, high availability

### Naming Conventions
```
Resource Group: rg-dataplatform-{env}
Storage Account: stdataplatform{env}{random}
Key Vault: kv-dataplatform-{env}
Databricks Workspace: dbw-dataplatform-{env}
Data Factory: adf-dataplatform-{env}
App Service: app-dataplatform-{env}
```

### Security Requirements
- All resources must use Managed Identity
- Network security groups for all subnets
- Private endpoints for storage and Key Vault
- RBAC with least privilege principle
- Secrets stored only in Key Vault

### Cost Optimization
- Auto-pause for dev Databricks clusters
- Lifecycle management for blob storage  
- Reserved instances for production
- Appropriate SKUs per environment

## Deliverables

### 1. Bicep Templates (`infrastructure/bicep/`)
- **main.bicep**: Master template orchestrating all modules
- **modules/**: Individual service templates
  - `storage.bicep` - Data Lake Storage Gen2
  - `databricks.bicep` - Databricks workspace  
  - `data-factory.bicep` - ADF pipelines
  - `app-service.bicep` - .NET API hosting
  - `key-vault.bicep` - Secrets management
  - `monitoring.bicep` - Azure Monitor setup

### 2. Environment Parameters (`infrastructure/bicep/environments/`)
- `dev.bicepparam` - Development configuration
- `staging.bicepparam` - Staging configuration  
- `prod.bicepparam` - Production configuration

### 3. Azure DevOps Pipelines (`.azure-devops/pipelines/`)
- `infrastructure-deploy.yml` - Infrastructure deployment
- `databricks-deploy.yml` - Notebooks deployment
- `api-deploy.yml` - .NET API deployment
- `powerbi-deploy.yml` - Power BI artifacts

### 4. Pipeline Templates (`.azure-devops/templates/`)
- `infrastructure.yml` - Reusable infrastructure steps
- `databricks.yml` - Databricks deployment template
- `dotnet-api.yml` - API build and deploy template
- `powerbi.yml` - Power BI deployment template

### 5. RBAC Configuration (`infrastructure/scripts/`)
- `deploy.ps1` - Main deployment script
- `setup-rbac.ps1` - Role assignments automation

## Quality Gates
Before marking tasks complete, ensure:

- ✅ All Bicep templates validate successfully
- ✅ Environment parameters properly configured  
- ✅ RBAC assignments follow least privilege
- ✅ Secrets stored only in Key Vault
- ✅ Cost optimization features enabled
- ✅ Monitoring and alerting configured
- ✅ Pipeline deployment tests pass

## Dependencies & Handoffs

### Prerequisites
- Azure subscription with sufficient quotas
- Azure DevOps organization setup
- Service principals for automation

### Provides to Other Agents
After successful deployment, create configuration files for other agents:

**File**: `infrastructure/outputs/azure-config.json`
```json
{
  "storageAccount": {
    "name": "stdataplatformdev12345",
    "connectionString": "@Microsoft.KeyVault(SecretUri=https://kv-dataplatform-dev.vault.azure.net/secrets/storage-connection)"
  },
  "databricks": {
    "workspaceUrl": "https://adb-123456789.azuredatabricks.net/",
    "resourceId": "/subscriptions/.../databricks-workspace"
  },
  "appService": {
    "url": "https://app-dataplatform-dev.azurewebsites.net",
    "managedIdentityClientId": "12345678-1234-1234-1234-123456789012"
  }
}
```

### Must Complete Before
Other agents depend on infrastructure completion for:
- **Databricks Agent**: Workspace URL and access tokens
- **API Agent**: App Service deployment target and Key Vault access  
- **Power BI Agent**: Data source connection strings

## Resource Specifications by Environment

### Development Environment
```bicep
param skuConfigs = {
  storage: 'Standard_LRS'
  appService: 'B1' 
  databricks: 'standard'
  dataFactory: 'Standard'
}
```

### Staging Environment  
```bicep
param skuConfigs = {
  storage: 'Standard_GRS'
  appService: 'S1'
  databricks: 'standard' 
  dataFactory: 'Standard'
}
```

### Production Environment
```bicep
param skuConfigs = {
  storage: 'Standard_RAGRS'
  appService: 'P1v2'
  databricks: 'premium'
  dataFactory: 'Standard'
}
```

## Monitoring & Alerting Configuration

### Key Metrics to Monitor
- **Storage**: Capacity usage, request rates, availability
- **Databricks**: Job success rates, cluster utilization  
- **App Service**: Response times, error rates, CPU/memory
- **Data Factory**: Pipeline success rates, execution duration

### Alert Rules
```json
{
  "storageAvailability": "< 99.9% for 5 minutes",
  "appServiceResponseTime": "> 1 second for 5 minutes", 
  "databricksJobFailures": "> 2 failures in 1 hour",
  "dataFactoryPipelineFailures": "Any failure immediately"
}
```

## Security Configuration

### Managed Identity Assignments
```powershell
# Storage Account Access
az role assignment create --assignee $databricksManagedIdentity \
  --role "Storage Blob Data Contributor" \
  --scope $storageAccountId

# Key Vault Access  
az role assignment create --assignee $appServiceManagedIdentity \
  --role "Key Vault Secrets User" \
  --scope $keyVaultId
```

### Network Security
- All services deployed in dedicated VNet
- Private endpoints for storage and Key Vault
- NSG rules allowing only required traffic
- Azure Firewall for egress filtering (production only)

## Troubleshooting Common Issues

### Bicep Deployment Failures
1. Check resource naming conflicts
2. Verify subscription quotas and limits
3. Ensure proper RBAC for deployment principal
4. Review parameter file syntax

### Pipeline Failures
1. Verify service connection permissions
2. Check agent pool availability  
3. Review variable group configurations
4. Validate artifact dependencies

For additional troubleshooting, see [docs/troubleshooting.md](../docs/troubleshooting.md)