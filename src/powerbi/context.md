# Power BI Agent Context

## Agent Role & Responsibilities
**Business Intelligence & Visualization Specialist**

**Scope**: Power BI datasets, reports, automated deployment

## Primary Tasks
1. Create Power BI datasets connected to Gold layer
2. Build interactive dashboards for Customer 360, Device Health, Revenue Analytics
3. Implement automated refresh and deployment  
4. Create PowerShell scripts for CI/CD integration
5. Set up usage monitoring and performance optimization

## Context Requirements

### READ FIRST
Review the following documentation before beginning implementation:
- [Data Schema Reference](../../docs/schema.md) - Gold layer tables and KPI definitions
- [Architecture Overview](../../docs/architecture.md) - Power BI integration architecture

### Data Sources
**Primary**: Connect to Gold layer Delta tables via Azure Databricks SQL Warehouse
**Secondary**: REST API endpoints for real-time data (optional)

### Dashboard Requirements
- **Customer 360**: Customer profile with key metrics, sales history, device status
- **Device Health**: Real-time device status grid, battery health, error analysis  
- **Revenue vs Device Health**: Correlation analysis between device issues and revenue

### Performance Standards
- **Dashboard load time**: <5 seconds for initial load
- **Refresh time**: <10 minutes for scheduled refresh
- **Concurrent users**: Support 100+ simultaneous users
- **Mobile responsive**: Optimized for tablets and phones

## Dashboard Specifications

### Customer 360 Dashboard

#### Key Visualizations
```dax
// Total Sales by Customer & Region
Total Sales = SUM(fact_sales[amount_usd])

Customer Sales by Region = 
CALCULATE(
    [Total Sales],
    ALLEXCEPT(dim_customer, dim_customer[region])
)

// Active Devices per Customer  
Active Device Count = 
CALCULATE(
    COUNT(dim_device[device_id]),
    dim_device[status] = "Healthy"
)

// Churn Risk Score
Churn Risk = 
VAR DeviceIssues = 
    CALCULATE(
        COUNT(dim_device[device_id]),
        dim_device[status] IN {"Needs Service", "Offline"}
    )
VAR LowUsage = 
    CALCULATE(
        COUNT(fact_device_usage[device_id]),
        fact_device_usage[usage_hours] < 2,
        fact_device_usage[event_date] >= TODAY() - 30
    )
RETURN
    SWITCH(
        TRUE(),
        DeviceIssues > 0 && LowUsage > 5, "High",
        DeviceIssues > 0 || LowUsage > 2, "Medium", 
        "Low"
    )
```

#### Page Layout
- **Header**: Customer summary cards (name, tier, lifetime value, device count)
- **Left Panel**: Customer filter and search  
- **Main Area**: Sales trends chart, device status grid
- **Right Panel**: Recent orders list, support tickets

### Device Health Dashboard

#### Key Visualizations  
```dax
// Device Uptime Percentage
Device Uptime = 
VAR TotalHours = 24 * COUNTROWS(CALENDAR(MIN(fact_device_usage[event_date]), MAX(fact_device_usage[event_date])))
VAR UsageHours = SUM(fact_device_usage[usage_hours])
RETURN
    DIVIDE(UsageHours, TotalHours, 0) * 100

// Battery Health Distribution
Battery Health Bucket = 
SWITCH(
    TRUE(),
    dim_device[battery_level] >= 80, "Excellent (80-100%)",
    dim_device[battery_level] >= 60, "Good (60-79%)", 
    dim_device[battery_level] >= 40, "Fair (40-59%)",
    dim_device[battery_level] >= 20, "Poor (20-39%)",
    "Critical (<20%)"
)

// Top Error Codes
Error Frequency = 
CALCULATE(
    COUNT(fact_device_usage[event_id]),
    fact_device_usage[error_code] <> "NONE"
)
```

#### Page Layout
- **Top Row**: KPI cards (total devices, healthy %, offline %, avg battery)
- **Middle Left**: Device status heatmap by region
- **Middle Right**: Battery health distribution donut chart
- **Bottom**: Top 10 error codes table with affected customer count

### Revenue vs Device Health Dashboard

#### Key Visualizations
```dax
// Revenue Impact of Device Issues
Revenue Impact = 
VAR CustomersWithIssues = 
    CALCULATETABLE(
        VALUES(dim_customer[customer_id]),
        dim_device[status] IN {"Needs Service", "Offline"}
    )
VAR RevenueWithIssues = 
    CALCULATE(
        [Total Sales],
        dim_customer[customer_id] IN CustomersWithIssues
    )
VAR TotalRevenue = [Total Sales]
RETURN
    DIVIDE(RevenueWithIssues, TotalRevenue, 0) * 100

// Renewal Rate by Device Status
Renewal Rate = 
VAR TotalCustomers = COUNT(dim_customer[customer_id])
VAR RenewedCustomers = 
    CALCULATE(
        COUNT(dim_customer[customer_id]),
        fact_sales[order_date] >= TODAY() - 365
    )
RETURN
    DIVIDE(RenewedCustomers, TotalCustomers, 0) * 100
```

## Data Model Design

### Relationships
```
dim_customer (1) ----< (many) fact_sales
dim_customer (1) ----< (many) dim_device  
dim_device (1) ----< (many) fact_device_usage
dim_product (1) ----< (many) fact_sales
dim_product (1) ----< (many) dim_device

Date Table (1) ----< (many) fact_sales[order_date]
Date Table (1) ----< (many) fact_device_usage[event_date]
```

### Calculated Tables
```dax
// Date dimension for time intelligence
Date = 
ADDCOLUMNS(
    CALENDAR(DATE(2020,1,1), TODAY()),
    "Year", YEAR([Date]),
    "Month", MONTH([Date]),
    "MonthName", FORMAT([Date], "MMMM"),
    "Quarter", "Q" & QUARTER([Date]),
    "Weekday", WEEKDAY([Date]),
    "WeekdayName", FORMAT([Date], "DDDD")
)

// Customer segments based on lifetime value
Customer Segments = 
ADDCOLUMNS(
    SUMMARIZE(dim_customer, dim_customer[customer_id], dim_customer[lifetime_value]),
    "Segment",
    SWITCH(
        TRUE(),
        [lifetime_value] >= 10000, "High Value",
        [lifetime_value] >= 2500, "Medium Value", 
        "Low Value"
    )
)
```

### Security Implementation (Row-Level Security)
```dax
// Regional access control - users only see their region's data
[Regional Access] = dim_customer[region] = USERPRINCIPALNAME()

// Manager access - regional managers see their regions
[Manager Access] = 
VAR UserEmail = USERPRINCIPALNAME()
VAR UserRegion = 
    LOOKUPVALUE(
        SecurityTable[region],
        SecurityTable[email], UserEmail
    )
RETURN
    dim_customer[region] = UserRegion
```

## Deployment Automation

### PowerShell Deployment Script
```powershell
# deploy-datasets.ps1
param(
    [Parameter(Mandatory=$true)]
    [string]$Environment,
    
    [Parameter(Mandatory=$true)]
    [string]$WorkspaceId,
    
    [string]$TenantId,
    [string]$ClientId,
    [string]$ClientSecret
)

# Authentication
$securePassword = ConvertTo-SecureString $ClientSecret -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential($ClientId, $securePassword)
Connect-PowerBIServiceAccount -Credential $credential -ServicePrincipal -Tenant $TenantId

# Deploy datasets
$datasets = @(
    "customer-360.pbit",
    "device-health.pbit", 
    "revenue-analytics.pbit"
)

foreach ($dataset in $datasets) {
    Write-Host "Deploying $dataset to $Environment environment..."
    
    # Upload .pbix file to workspace
    $import = New-PowerBIReport -Path "datasets\$dataset" -WorkspaceId $WorkspaceId -Name "$dataset-$Environment"
    
    # Update data source connections
    $datasetId = $import.Id
    $connectionString = Get-EnvironmentConnectionString -Environment $Environment
    
    Set-PowerBIDataset -DatasetId $datasetId -ConnectionString $connectionString
    
    # Configure refresh schedule
    Set-PowerBIDatasetRefresh -DatasetId $datasetId -Schedule @{
        days = @("Monday", "Tuesday", "Wednesday", "Thursday", "Friday")
        times = @("08:00", "14:00", "20:00")
        enabled = $true
    }
    
    Write-Host "Successfully deployed $dataset"
}

Write-Host "All datasets deployed successfully to $Environment"
```

### Dataset Refresh Automation
```powershell
# refresh-datasets.ps1
param(
    [string[]]$DatasetIds,
    [int]$MaxWaitMinutes = 60
)

foreach ($datasetId in $DatasetIds) {
    Write-Host "Starting refresh for dataset $datasetId..."
    
    # Trigger refresh
    Invoke-PowerBIRestMethod -Url "datasets/$datasetId/refreshes" -Method Post
    
    # Monitor refresh status
    $timeout = (Get-Date).AddMinutes($MaxWaitMinutes)
    do {
        Start-Sleep -Seconds 30
        $refreshStatus = Invoke-PowerBIRestMethod -Url "datasets/$datasetId/refreshes?`$top=1" -Method Get | ConvertFrom-Json
        $status = $refreshStatus.value[0].status
        
        Write-Host "Refresh status: $status"
        
        if ($status -eq "Completed") {
            Write-Host "Refresh completed successfully for dataset $datasetId"
            break
        }
        elseif ($status -eq "Failed") {
            Write-Error "Refresh failed for dataset $datasetId"
            break
        }
    } while ((Get-Date) -lt $timeout)
    
    if ((Get-Date) -ge $timeout) {
        Write-Warning "Refresh timeout reached for dataset $datasetId"
    }
}
```

## Performance Optimization

### Data Model Optimization
```dax
// Aggregation tables for large fact tables
Sales Summary = 
SUMMARIZECOLUMNS(
    dim_customer[region],
    dim_customer[subscription_tier],
    'Date'[Year],
    'Date'[Month],
    "Total Sales", SUM(fact_sales[amount_usd]),
    "Order Count", COUNT(fact_sales[order_id]),
    "Unique Customers", DISTINCTCOUNT(fact_sales[customer_id])
)

// Calculated columns for commonly filtered attributes
Customer Tier Numeric = 
SWITCH(
    dim_customer[subscription_tier],
    "Premium", 3,
    "Standard", 2, 
    "Basic", 1,
    0
)
```

### Query Optimization
- Use SUMMARIZECOLUMNS instead of SUMMARIZE for better performance
- Implement bidirectional filtering carefully to avoid circular dependencies
- Use KEEPFILTERS() to maintain filter context in complex calculations
- Create aggregation tables for frequently accessed large datasets

## Deliverables

### 1. Power BI Templates (`src/powerbi/datasets/`)
- **customer-360.pbit**: Customer analytics template
- **device-health.pbit**: IoT device monitoring template  
- **revenue-analytics.pbit**: Revenue correlation analysis template

Each template includes:
- Optimized data model with relationships
- Pre-built measures and calculated columns
- Formatted visualizations with corporate branding
- Row-level security configuration

### 2. Deployment Scripts (`src/powerbi/scripts/`)
- **deploy-datasets.ps1**: Automated dataset deployment
- **refresh-datasets.ps1**: Refresh automation and monitoring
- **configure-security.ps1**: RLS and permissions setup

### 3. Documentation (`src/powerbi/`)
- **README.md**: Setup and deployment instructions
- **user-guide.md**: End-user documentation for each dashboard
- **admin-guide.md**: Administration and maintenance procedures

### 4. Testing (`src/powerbi/tests/`)
- **data-validation.ps1**: Automated data quality checks
- **performance-test.ps1**: Dashboard load time validation
- **security-test.ps1**: RLS verification scripts

## Quality Gates

Before marking deliverables complete:

- ✅ Dashboards load in under 5 seconds consistently
- ✅ Data model optimized with proper relationships and aggregations
- ✅ Row-level security implemented and tested
- ✅ Mobile responsive design validated on tablet/phone
- ✅ Automated deployment working across environments  
- ✅ User documentation complete with screenshots
- ✅ Performance metrics captured and within SLA targets

## Dependencies & Handoffs

### Prerequisites  
- **Databricks Agent**: Gold layer tables populated with sample data
- **API Agent** (optional): REST endpoints for real-time integration
- **Infrastructure Agent**: Power BI Premium workspace provisioned

### Environment Configuration
```json
{
  "development": {
    "databricksUrl": "https://adb-dev-12345.azuredatabricks.net/",
    "warehouseId": "dev-sql-warehouse",
    "workspaceId": "12345678-dev-workspace"
  },
  "production": {
    "databricksUrl": "https://adb-prod-67890.azuredatabricks.net/", 
    "warehouseId": "prod-sql-warehouse",
    "workspaceId": "87654321-prod-workspace"
  }
}
```

## Monitoring & Usage Analytics

### Power BI Usage Tracking
```powershell
# Get usage analytics
$usageData = Invoke-PowerBIRestMethod -Url "admin/reports/getReportUsersAsAdmin" -Method Get
$usage = $usageData | ConvertFrom-Json

# Track key metrics
$metrics = @{
    "daily_active_users" = ($usage.value | Where-Object { $_.lastAccessedDate -ge (Get-Date).AddDays(-1) }).Count
    "weekly_active_users" = ($usage.value | Where-Object { $_.lastAccessedDate -ge (Get-Date).AddDays(-7) }).Count
    "monthly_active_users" = ($usage.value | Where-Object { $_.lastAccessedDate -ge (Get-Date).AddDays(-30) }).Count
}

# Send metrics to Azure Monitor
Send-AzureMonitorMetric -Metrics $metrics -ResourceId $PowerBIWorkspaceResourceId
```

### Performance Monitoring  
```dax
// Dashboard performance measures
Page Load Time = 
VAR StartTime = NOW()
VAR EndTime = NOW()
RETURN
    (EndTime - StartTime) * 24 * 60 * 60 // Convert to seconds

Query Duration = 
CALCULATE(
    MAX('Query Performance'[Duration]),
    'Query Performance'[QueryType] = "Dashboard Load"
)
```

## Troubleshooting Common Issues

### Data Connection Issues
1. **Databricks connectivity**: Verify SQL warehouse is running and accessible
2. **Authentication failures**: Check service principal permissions on Databricks workspace
3. **Timeout errors**: Implement query optimization or data model aggregations

### Performance Issues  
1. **Slow dashboard loads**: Review data model relationships and eliminate unnecessary columns
2. **Refresh failures**: Check for data quality issues in Gold layer tables
3. **Memory errors**: Implement incremental refresh for large datasets

### Security Issues
1. **RLS not working**: Validate security table relationships and DAX filter expressions
2. **User access denied**: Check Azure AD group memberships and Power BI workspace permissions
3. **Data exposure**: Audit RLS implementation with test accounts from different regions

For additional troubleshooting, see [docs/troubleshooting.md](../../docs/troubleshooting.md)