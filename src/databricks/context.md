# Databricks Agent Context

## Agent Role & Responsibilities  
**Data Engineering & ETL Specialist**

**Scope**: PySpark notebooks, Delta Lake, Data pipelines

## Primary Tasks
1. Create bronze-to-silver-to-gold ETL notebooks
2. Implement data quality checks and schema validation
3. Set up streaming ingestion from IoT Hub
4. Configure Databricks jobs with proper scheduling
5. Create comprehensive test suites

## Context Requirements

### READ FIRST
Review the following documentation before beginning implementation:
- [Data Schema Reference](../../docs/schema.md) - Complete data layer definitions
- [Architecture Overview](../../docs/architecture.md) - Data flow and processing architecture

### Data Architecture
**Bronze (raw) → Silver (cleaned) → Gold (analytics-ready)**

### Framework & Tools
- **PySpark** with Delta Lake for ACID transactions
- **Great Expectations** for data quality validation
- **MLflow** for experiment tracking (future ML workflows)
- **Databricks Workflows** for job orchestration

### Quality Standards
- Data validation and schema enforcement
- Deduplication based on primary keys
- Error handling with dead letter queues
- Data lineage tracking

## Data Quality Rules

### Bronze → Silver Transformation Rules
```python
# Data quality checks to implement
QUALITY_RULES = {
    "customers": {
        "required_fields": ["customer_id", "email"],
        "email_validation": r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
        "deduplication_key": "customer_id"
    },
    "sales_orders": {
        "required_fields": ["order_id", "customer_id", "amount"],
        "amount_validation": "amount > 0",
        "currency_standardization": "USD",
        "deduplication_key": "order_id"  
    },
    "device_events": {
        "temperature_range": "temperature BETWEEN -50 AND 100",
        "battery_range": "battery_level BETWEEN 0 AND 100",
        "error_code_standardization": "COALESCE(error_code, 'NONE')"
    }
}
```

### Schema Evolution Strategy
- Handle new columns gracefully with schema merging
- Version all schema changes in Git
- Maintain backward compatibility
- Document schema evolution in data catalog

## Performance Standards

### Processing Time SLAs
- **Bronze ingestion**: <30min for daily batch
- **Silver transformation**: <15min per table  
- **Gold aggregation**: <10min for all KPIs
- **Streaming latency**: <5min for IoT events

### Optimization Techniques
```python
# Delta Lake optimizations to implement
def optimize_tables():
    """Optimize Delta tables for query performance"""
    
    # Z-order optimization for frequently filtered columns
    spark.sql("OPTIMIZE bronze_device_events ZORDER BY (device_id, event_time)")
    
    # Vacuum old files (7 day retention)
    spark.sql("VACUUM bronze_device_events RETAIN 168 HOURS")
    
    # Table statistics for cost-based optimization
    spark.sql("ANALYZE TABLE silver_customers COMPUTE STATISTICS FOR ALL COLUMNS")
```

## Error Handling Strategy

### Retry Logic
- Retry failed transformations 3 times with exponential backoff
- Dead letter queue for persistently corrupted data
- Alerting on job failures via Azure Monitor
- Manual data repair procedures documented

### Data Lineage Tracking
```python
# Implement data lineage tracking
def track_data_lineage(source_table, target_table, transformation_type, record_count):
    """Track data transformation lineage"""
    lineage_entry = {
        "source_table": source_table,
        "target_table": target_table, 
        "transformation_type": transformation_type,
        "record_count": record_count,
        "processed_at": datetime.now(),
        "job_run_id": dbutils.notebook.entry_point.getDbutils().notebook().getContext().runId()
    }
    # Write to lineage tracking table
```

## Deliverables

### 1. ETL Notebooks (`src/databricks/notebooks/`)

#### Bronze Ingestion (`bronze-ingestion/`)
- `ingest-sales-data.py` - CSV file ingestion from Blob Storage
- `ingest-customer-data.py` - CRM data integration  
- `ingest-iot-stream.py` - Real-time IoT Hub streaming

**Template Structure**:
```python
# Standard notebook header for all bronze ingestion
from pyspark.sql import SparkSession
from pyspark.sql.types import *
from delta.tables import DeltaTable
import great_expectations as ge

# Configuration
spark = SparkSession.builder.appName("BronzeIngestion").getOrCreate()
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")

# Schema definitions
SALES_ORDER_SCHEMA = StructType([
    StructField("order_id", StringType(), False),
    StructField("customer_id", StringType(), False),
    # ... complete schema definition
])
```

#### Silver Transformation (`silver-transformation/`)
- `clean-sales-data.py` - Data quality and standardization
- `clean-customer-data.py` - Email validation and region standardization
- `clean-device-events.py` - IoT data cleansing and validation

#### Gold Aggregation (`gold-aggregation/`)
- `create-dimensions.py` - Build dimension tables (customer, product, device)
- `create-facts.py` - Build fact tables (sales, device usage)
- `calculate-kpis.py` - Calculate business metrics and KPIs

### 2. Job Configurations (`src/databricks/jobs/`)

#### Workflow Definitions
- `bronze-to-silver-job.json` - Daily batch processing
- `silver-to-gold-job.json` - Business logic transformations
- `full-pipeline-job.json` - End-to-end pipeline orchestration

**Example Job Configuration**:
```json
{
  "name": "bronze-to-silver-pipeline",
  "job_clusters": [
    {
      "job_cluster_key": "etl-cluster",
      "new_cluster": {
        "spark_version": "13.3.x-scala2.12",
        "node_type_id": "Standard_DS3_v2",
        "num_workers": 2,
        "auto_termination_minutes": 30
      }
    }
  ],
  "tasks": [
    {
      "task_key": "clean-sales-data",
      "notebook_task": {
        "notebook_path": "/silver-transformation/clean-sales-data"
      },
      "job_cluster_key": "etl-cluster"
    }
  ]
}
```

### 3. Test Suite (`src/databricks/tests/`)

#### Unit Tests
- `test_bronze_ingestion.py` - Ingestion logic validation
- `test_silver_transformation.py` - Data quality rule testing  
- `test_gold_aggregation.py` - Business logic verification

**Test Framework**:
```python
import pytest
from pyspark.sql import SparkSession
from pyspark.sql.types import *

@pytest.fixture(scope="session")
def spark():
    return SparkSession.builder.appName("pytest").getOrCreate()

def test_customer_deduplication(spark):
    """Test customer deduplication logic"""
    # Create test data with duplicates
    test_data = [
        ("CUST_001", "john@email.com", "2024-01-01"),
        ("CUST_001", "john@email.com", "2024-01-02")  # Duplicate
    ]
    
    # Run deduplication logic
    result = deduplicate_customers(spark.createDataFrame(test_data, schema))
    
    # Assert single record returned
    assert result.count() == 1
```

## Streaming Data Configuration

### IoT Hub Integration
```python
# Structured Streaming configuration for IoT events
def setup_iot_streaming():
    """Configure streaming from IoT Hub to Delta Lake"""
    
    stream = (spark
        .readStream
        .format("eventhubs")
        .option("eventhubs.connectionString", get_secret("iot-hub-connection"))
        .option("eventhubs.consumerGroup", "databricks-consumer")
        .load()
    )
    
    # Parse JSON events and write to Bronze
    parsed_stream = (stream
        .select(F.col("body").cast("string").alias("json_data"))
        .select(F.from_json("json_data", device_event_schema).alias("event"))
        .select("event.*")
    )
    
    # Write to Delta table with checkpointing
    query = (parsed_stream
        .writeStream
        .format("delta")
        .outputMode("append")
        .option("checkpointLocation", "/mnt/checkpoints/device-events")
        .table("bronze_device_events")
    )
    
    return query
```

## Quality Gates

Before marking deliverables complete:

- ✅ All transformations handle schema drift gracefully
- ✅ Data quality checks implemented on every layer
- ✅ Proper error logging and monitoring configured
- ✅ Delta Lake optimize and vacuum scheduled  
- ✅ Unit tests achieve >90% code coverage
- ✅ Integration tests pass with sample data
- ✅ Performance benchmarks meet SLA requirements
- ✅ Documentation updated with data lineage

## Dependencies & Handoffs

### Prerequisites
- Infrastructure Agent: Databricks workspace and storage accounts deployed
- Sample data available in `data/samples/` directory

### Provides to Other Agents
**File**: `src/databricks/outputs/gold-schema.json`
```json
{
  "dimensions": {
    "dim_customer": {
      "table": "gold.dim_customer",
      "primary_key": "customer_id", 
      "columns": ["customer_id", "name", "email", "region", "lifetime_value"]
    }
  },
  "facts": {
    "fact_sales": {
      "table": "gold.fact_sales", 
      "grain": "order_id",
      "measures": ["amount_usd"],
      "dimensions": ["customer_id", "product_id"]
    }
  }
}
```

### Success Metrics
- **Data Quality**: <5% error rate across all transformations
- **Performance**: Meet processing time SLAs consistently  
- **Reliability**: >95% job success rate
- **Coverage**: >90% test coverage for all notebooks

## Troubleshooting Guide

### Common Issues
1. **Schema Evolution Failures**: Enable auto merge schema option
2. **Memory Issues**: Increase driver/executor memory, enable adaptive query execution
3. **Checkpoint Corruption**: Clear checkpoint directory and restart stream
4. **Delta Log Issues**: Run VACUUM and OPTIMIZE operations

### Monitoring Queries
```sql
-- Check data quality metrics
SELECT 
    table_name,
    COUNT(*) as total_records,
    SUM(CASE WHEN data_quality_flag = false THEN 1 ELSE 0 END) as error_count
FROM silver_data_quality_log 
WHERE process_date = current_date()
GROUP BY table_name;

-- Monitor processing times  
SELECT 
    job_name,
    AVG(duration_minutes) as avg_duration,
    MAX(duration_minutes) as max_duration
FROM job_execution_log 
WHERE run_date >= current_date() - 7
GROUP BY job_name;
```