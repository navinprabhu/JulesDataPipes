# Data Schema Reference

## Data Domains

The platform processes three main data domains:

- **Sales Data (ERP/CRM)** – Orders, Subscriptions, Revenue
- **Customer Data (CRM)** – Profiles, Segments, Support  
- **IoT Device Data (Telemetry)** – Usage, Health, Errors

## Bronze Layer (Raw Landing Zone)

**Purpose**: Store raw data as-is with no transformations for audit and reprocessing.

### bronze_sales_orders
```sql
CREATE TABLE bronze_sales_orders (
  order_id STRING,
  customer_id STRING,
  product_id STRING, 
  order_date TIMESTAMP,
  amount DECIMAL(10,2),
  currency STRING,
  payment_status STRING
) USING DELTA
```

### bronze_customers
```sql
CREATE TABLE bronze_customers (
  customer_id STRING,
  first_name STRING,
  last_name STRING,
  email STRING,
  region STRING,
  subscription_tier STRING,
  signup_date TIMESTAMP
) USING DELTA
```

### bronze_devices
```sql
CREATE TABLE bronze_devices (
  device_id STRING,
  customer_id STRING,
  product_id STRING,
  purchase_date TIMESTAMP
) USING DELTA
```

### bronze_device_events (IoT Stream)
```sql
CREATE TABLE bronze_device_events (
  event_id STRING,
  device_id STRING,
  event_time TIMESTAMP,
  temperature DOUBLE,
  battery_level INT,
  error_code STRING,
  usage_hours DOUBLE
) USING DELTA
PARTITIONED BY (DATE(event_time))
```

## Silver Layer (Cleaned & Standardized)

**Purpose**: Apply data quality rules, deduplication, standard formats, and basic joins.

### Data Quality Rules Applied:
- Remove duplicates based on primary keys
- Validate required fields not null  
- Standardize date formats to ISO 8601
- Currency conversion to USD
- Email validation and cleansing

### silver_sales_orders
```sql
CREATE TABLE silver_sales_orders (
  order_id STRING,
  customer_id STRING,
  product_id STRING,
  order_date TIMESTAMP,
  amount_usd DECIMAL(10,2),  -- Standardized currency
  payment_status STRING,
  load_timestamp TIMESTAMP,
  CONSTRAINT pk_silver_sales PRIMARY KEY (order_id)
) USING DELTA
```

**Transformations**:
- ✅ Currency standardized to USD
- ✅ Duplicates removed by order_id
- ✅ Invalid amounts filtered out

### silver_customers  
```sql
CREATE TABLE silver_customers (
  customer_id STRING,
  full_name STRING,           -- Concatenated first + last name
  email STRING,               -- Validated and lowercased
  region STRING,              -- Standardized region codes
  subscription_tier STRING,
  signup_date TIMESTAMP,
  is_active BOOLEAN,
  CONSTRAINT pk_silver_customers PRIMARY KEY (customer_id)
) USING DELTA
```

**Transformations**:
- ✅ Email validation and standardization
- ✅ Region codes unified (US, EU, APAC)
- ✅ Active status derived from recent activity

### silver_devices
```sql
CREATE TABLE silver_devices (
  device_id STRING,
  customer_id STRING,
  product_id STRING, 
  purchase_date TIMESTAMP,
  status STRING,              -- Derived: "Active", "Inactive", "Maintenance"
  CONSTRAINT pk_silver_devices PRIMARY KEY (device_id)
) USING DELTA
```

**Transformations**:
- ✅ One record per device (deduplicated)
- ✅ Status derived from recent telemetry

### silver_device_events
```sql
CREATE TABLE silver_device_events (
  event_id STRING,
  device_id STRING,
  event_time TIMESTAMP,
  temperature DOUBLE,
  battery_level INT,          -- Normalized to 0-100
  error_code STRING,          -- NULL replaced with "NONE"
  usage_hours DOUBLE,
  is_valid BOOLEAN,          -- Data quality flag
  CONSTRAINT pk_silver_events PRIMARY KEY (event_id)
) USING DELTA
PARTITIONED BY (DATE(event_time))
```

**Transformations**:
- ✅ Corrupted sensor readings filtered out
- ✅ Battery level normalized to 0-100 range
- ✅ Error codes standardized ("NONE" for nulls)

## Gold Layer (Analytics-Ready Star Schema)

**Purpose**: Business-ready dimensional model optimized for reporting and analytics.

### Dimension Tables

#### dim_customer
```sql
CREATE TABLE dim_customer (
  customer_id STRING,
  name STRING,
  email STRING,
  region STRING,
  subscription_tier STRING,
  signup_date TIMESTAMP,
  lifetime_value DECIMAL(12,2), -- Calculated: total sales per customer
  device_count INT,             -- Number of active devices
  last_order_date TIMESTAMP,
  customer_segment STRING,      -- High/Medium/Low value
  CONSTRAINT pk_dim_customer PRIMARY KEY (customer_id)
) USING DELTA
```

#### dim_product  
```sql
CREATE TABLE dim_product (
  product_id STRING,
  product_name STRING,
  category STRING,
  price DECIMAL(10,2),
  launch_date TIMESTAMP,
  is_active BOOLEAN,
  CONSTRAINT pk_dim_product PRIMARY KEY (product_id)
) USING DELTA
```

#### dim_device
```sql
CREATE TABLE dim_device (
  device_id STRING,
  customer_id STRING,
  product_id STRING,
  purchase_date TIMESTAMP,
  last_event_time TIMESTAMP,
  status STRING,              -- "Healthy", "Needs Service", "Offline"
  avg_daily_usage DOUBLE,     -- Average usage hours per day
  health_score INT,           -- 0-100 derived from telemetry
  CONSTRAINT pk_dim_device PRIMARY KEY (device_id)
) USING DELTA
```

### Fact Tables

#### fact_sales
```sql
CREATE TABLE fact_sales (
  order_id STRING,
  customer_id STRING,        -- FK to dim_customer
  product_id STRING,         -- FK to dim_product  
  order_date TIMESTAMP,
  amount_usd DECIMAL(10,2),
  payment_status STRING,
  order_year INT,
  order_month INT,
  order_day INT,
  CONSTRAINT pk_fact_sales PRIMARY KEY (order_id)
) USING DELTA
PARTITIONED BY (order_year, order_month)
```

#### fact_device_usage
```sql
CREATE TABLE fact_device_usage (
  event_id STRING,
  device_id STRING,          -- FK to dim_device
  event_date DATE,
  event_time TIMESTAMP,
  usage_hours DOUBLE,
  battery_level INT,
  error_code STRING,
  temperature DOUBLE,
  event_year INT,
  event_month INT,
  CONSTRAINT pk_fact_usage PRIMARY KEY (event_id)
) USING DELTA  
PARTITIONED BY (event_year, event_month)
```

## Key Performance Indicators (KPIs)

### Customer 360 Dashboard
- **Total Sales by Customer & Region**: `SUM(fact_sales.amount_usd) GROUP BY dim_customer.region`
- **Active Devices per Customer**: `COUNT(dim_device.device_id) WHERE status = 'Healthy'`
- **Churn Risk**: Customers with failing devices + low usage in last 30 days

### Device Health Dashboard  
- **Device Uptime**: `AVG(usage_hours) / 24 * 100` percentage per day
- **Battery Health Distribution**: Histogram of `battery_level` across all devices
- **Top Error Codes**: `COUNT(*) GROUP BY error_code ORDER BY count DESC LIMIT 10`

### Revenue vs Device Health
- **Sales Correlation**: Revenue trends vs. device error rates by month
- **Renewal Rate**: Subscription renewals by device status (Healthy vs. Needs Service)

## API Data Models

### Customer Profile Response
```json
{
  "customerId": "CUST_12345",
  "name": "John Smith", 
  "email": "john.smith@email.com",
  "region": "US",
  "subscriptionTier": "Premium",
  "lifetimeValue": 15750.50,
  "deviceCount": 3,
  "lastOrderDate": "2024-08-15T10:30:00Z",
  "devices": [
    {
      "deviceId": "DEV_67890",
      "status": "Healthy", 
      "healthScore": 95,
      "lastSeen": "2024-08-18T14:22:00Z"
    }
  ],
  "recentOrders": [
    {
      "orderId": "ORD_111",
      "amount": 299.99,
      "orderDate": "2024-08-15T10:30:00Z"
    }
  ]
}
```

### Device Alerts Response
```json
{
  "alerts": [
    {
      "deviceId": "DEV_12345",
      "customerId": "CUST_67890", 
      "alertType": "Low Battery",
      "severity": "Medium",
      "batteryLevel": 15,
      "lastSeen": "2024-08-18T14:45:00Z",
      "recommendedAction": "Schedule maintenance"
    }
  ],
  "totalCount": 23,
  "generatedAt": "2024-08-18T15:00:00Z"
}
```

## Data Lineage

```
Source Systems → Bronze (Raw) → Silver (Clean) → Gold (Analytics) → Consumption
     ↓               ↓              ↓               ↓                ↓
[ERP/CRM/IoT] → [Delta Tables] → [Validated] → [Star Schema] → [Power BI/API]
     ↓               ↓              ↓               ↓                ↓
- CSV Files     - Audit Trail   - Quality Rules - Business Logic - Dashboards  
- JSON Events   - Schema on Read - Deduplication - KPI Calculation - REST APIs
- SQL Queries   - Partitioning   - Standardization - Dimensional Model - Reports
```