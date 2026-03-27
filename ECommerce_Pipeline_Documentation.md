# ECommerce Orders Medallion Architecture Flowchart

## Medallion Architecture Layers & Tables

mermaid
flowchart TD
    subgraph S3[Raw Data: S3 Bucket]
        S3CSV[Raw CSVs]
    end

    S3CSV --> Bronze

    subgraph Bronze["BRONZE LAYER\n(01_bronze)"]
        B_orders["orders"]
        B_customers["customers"]
        B_products["products"]
        B_sellers["sellers"]
        B_order_items["order_items"]
        B_order_payments["order_payments"]
        B_order_reviews["order_reviews"]
        B_geolocation["geolocation"]
        B_category_translation["product_category_name_translation"]
    end

    Bronze --> Silver

    subgraph Silver["SILVER LAYER\n(02_silver)"]
        S_orders["orders"]
        S_customers["customers"]
        S_products["products"]
        S_sellers["sellers"]
        S_order_items["order_items"]
        S_order_payments["order_payments"]
        S_order_reviews["order_reviews"]
        S_geolocation["geolocation"]
        S_category_translation["product_category_name_translation"]
    end

    Silver --> Gold

    subgraph Gold["GOLD LAYER\n(03_gold)"]
        subgraph Dimensions["Dimensions"]
            G_dim_customer["dim_customer"]
            G_dim_date["dim_date"]
            G_dim_geolocation["dim_geolocation"]
            G_dim_products["dim_products"]
            G_dim_sellers["dim_sellers"]
        end
        subgraph Facts["Facts"]
            G_fact_orders["fact_orders"]
            G_fact_order_items["fact_order_items"]
            G_fact_payments["fact_payments"]
            G_fact_reviews["fact_reviews"]
        end
    end

    Gold --> KPI

    subgraph KPI["KPI LAYER\n(03_gold)"]
        K_aov["kpi_aov"]
        K_avg_del_time["kpi_avg_del_time"]
        K_clv["kpi_clv"]
        K_late_del["kpi_late_delivery_percentage"]
        K_fulfillment["kpi_order_fulfillment_rate"]
        K_low_rated["kpi_percentage_low_rated_orders"]
        K_revenue_month["kpi_revenue_by_month"]
        K_top_products["kpi_top10_products_by_revenue"]
        K_top_sellers["kpi_top10_sellers_by_revenue"]
    end

    %% Bronze to Silver
    B_orders --> S_orders
    B_customers --> S_customers
    B_products --> S_products
    B_sellers --> S_sellers
    B_order_items --> S_order_items
    B_order_payments --> S_order_payments
    B_order_reviews --> S_order_reviews
    B_geolocation --> S_geolocation
    B_category_translation --> S_category_translation

    %% Silver to Gold Dimensions
    S_customers --> G_dim_customer
    S_orders --> G_dim_date
    S_geolocation --> G_dim_geolocation
    S_products --> G_dim_products
    S_sellers --> G_dim_sellers

    %% Silver to Gold Facts
    S_orders --> G_fact_orders
    S_order_items --> G_fact_order_items
    S_order_payments --> G_fact_payments
    S_order_reviews --> G_fact_reviews

    %% Gold Dimensions to Facts
    G_dim_customer --> G_fact_orders
    G_dim_date --> G_fact_orders
    G_dim_products --> G_fact_order_items
    G_dim_sellers --> G_fact_order_items
    G_dim_geolocation --> G_fact_orders

    %% Gold Facts to KPI
    G_fact_orders --> K_aov
    G_fact_orders --> K_fulfillment
    G_fact_orders --> K_revenue_month
    G_fact_orders --> K_avg_del_time
    G_fact_orders --> K_late_del
    G_fact_orders --> K_low_rated
    G_fact_orders --> K_clv
    G_fact_order_items --> K_top_products
    G_fact_order_items --> K_top_sellers


---

## Layer Table Details

### Bronze Layer (01_bronze)
- `orders`
- `customers`
- `products`
- `sellers`
- `order_items`
- `order_payments`
- `order_reviews`
- `geolocation`
- `product_category_name_translation`

### Silver Layer (02_silver)
- `orders`
- `customers`
- `products`
- `sellers`
- `order_items`
- `order_payments`
- `order_reviews`
- `geolocation`
- `product_category_name_translation`

### Gold Layer (03_gold)
#### Dimensions
- `dim_customer`
- `dim_date`
- `dim_geolocation`
- `dim_products`
- `dim_sellers`

#### Facts
- `fact_orders`
- `fact_order_items`
- `fact_payments`
- `fact_reviews`

### KPI Layer (03_gold)
- `kpi_aov`
- `kpi_avg_del_time`
- `kpi_clv`
- `kpi_late_delivery_percentage`
- `kpi_order_fulfillment_rate`
- `kpi_percentage_low_rated_orders`
- `kpi_revenue_by_month`
- `kpi_top10_products_by_revenue`
- `kpi_top10_sellers_by_revenue`

---

**Schema Naming Convention**:
- Bronze: `ecommerce_orders.01_bronze.<table_name>`
- Silver: `ecommerce_orders.02_silver.<table_name>`
- Gold: `ecommerce_orders.03_gold.dim_<name>`,

## 4. Silver Layer - Data Transformation

### Objective
Cleanse, standardize, and validate Bronze data for analytical use.

### Silver Notebooks (9)

1. `nb_transform_orders` - Transform orders
2. `nb_transform_customers` - Transform customers
3. `nb_transform_products` - Transform products  
4. `nb_transform_sellers` - Transform sellers
5. `nb_transform_order_items` - Transform order items
6. `nb_transform_order_payments` - Transform & aggregate payments
7. `nb_transform_order_reviews` - Transform reviews
8. `nb_transform_geolocation` - Transform location data
9. `nb_transform_product_category_translation` - Transform categories

### Key Transformations

#### Orders
- Cast timestamp columns to proper types
- Standardize order_status values
- Calculate delivery_days

```python
from pyspark.sql.functions import col, to_timestamp, datediff

orders_silver = spark.table("ecommerce_orders.01_bronze.orders") \
    .withColumn("order_purchase_timestamp", to_timestamp("order_purchase_timestamp")) \
    .withColumn("delivery_days", 
                datediff(col("order_delivered_timestamp"), 
                        col("order_purchase_timestamp")))
```

#### Order Payments (Aggregation)
```sql
CREATE OR REPLACE TABLE ecommerce_orders.02_silver.order_payments AS
SELECT 
    order_id,
    COLLECT_SET(payment_type) AS payment_types,
    SUM(payment_value) AS payment_value
FROM ecommerce_orders.01_bronze.order_payments
GROUP BY order_id
```

### Best Practices
✅ Apply consistent data types  
✅ Implement business rules and validations  
✅ Add data quality checks  
✅ Document transformation logic  
✅ Add `processed_timestamp` column  

---

## 5. Gold Layer - Dimensional Modeling

### 🔴 CRITICAL DEPENDENCY RULE
**Fact tables MUST depend on dimension tables!**
- Dimensions must be created FIRST
- Facts reference dimension keys
- Job orchestration must enforce: **Dimensions → Facts**

---

### Dimension Notebooks (5)

#### 1. `nb_dim_customer`
```python
dim_customer = spark.sql('''
    SELECT DISTINCT
        customer_id,
        customer_unique_id,
        customer_zip_code_prefix AS zip_code,
        customer_city AS city,
        customer_state AS state
    FROM ecommerce_orders.02_silver.customers
''')

dim_customer.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("ecommerce_orders.03_gold.dim_customer")
```

#### 2. `nb_dim_date`
Generate date dimension for 2016-2025 with year, month, quarter, day attributes.

#### 3. `nb_dim_geolocation`
Deduplicate and aggregate location data.

#### 4. `nb_dim_products`
Join with category translations, handle missing categories.

#### 5. `nb_dim_sellers`
Extract unique seller information.

---

### Fact Notebooks (4)

#### 1. `nb_fact_orders` ⚠️ Depends on: dim_customer, dim_date
```python
fact_orders = spark.sql('''
    SELECT 
        o.order_id,
        o.customer_id,
        DATE(o.order_purchase_timestamp) AS order_date,
        o.order_status,
        o.order_purchase_timestamp,
        o.order_delivered_timestamp,
        DATEDIFF(o.order_delivered_timestamp, o.order_purchase_timestamp) AS delivery_days,
        p.payment_value AS order_value
    FROM ecommerce_orders.02_silver.orders o
    LEFT JOIN ecommerce_orders.02_silver.order_payments p 
        ON o.order_id = p.order_id
    INNER JOIN ecommerce_orders.03_gold.dim_customer c 
        ON o.customer_id = c.customer_id
''')
```

#### 2. `nb_fact_order_items` ⚠️ Depends on: dim_products, dim_sellers, fact_orders
Must run AFTER fact_orders.

#### 3. `nb_fact_payments` ⚠️ Depends on: fact_orders
Must run AFTER fact_orders.

#### 4. `nb_fact_reviews` ⚠️ Depends on: fact_orders  
Must run AFTER fact_orders.

---

## 6. KPI Layer - Business Metrics

### KPI 1: Average Order Value
```sql
CREATE OR REPLACE TABLE ecommerce_orders.03_gold.kpi_aov AS
SELECT AVG(payment_value) AS avg_order_value
FROM ecommerce_orders.03_gold.fact_payments;
```

### KPI 2: Order Fulfillment Rate
```sql
CREATE OR REPLACE TABLE ecommerce_orders.03_gold.kpi_order_fulfillment_rate AS
SELECT 
    COUNT(CASE WHEN order_status = 'delivered' THEN 1 END) * 100.0 / COUNT(*) AS fulfillment_rate
FROM ecommerce_orders.03_gold.fact_orders;
```

### KPI 3: Revenue by Month
```sql
CREATE OR REPLACE TABLE ecommerce_orders.03_gold.kpi_revenue_by_month AS
SELECT 
    DATE_TRUNC('month', order_date) AS month,
    SUM(order_value) AS monthly_revenue,
    COUNT(DISTINCT order_id) AS order_count
FROM ecommerce_orders.03_gold.fact_orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

*(Remaining 6 KPIs follow similar patterns)*

---

## 7. Job Creation Guide

### Create 5 Separate Jobs

#### Job 1: Bronze Ingestion
- **Name**: `Job_01_Bronze_Ingestion`
- **Tasks**: 9 tasks (one per bronze notebook)
- **Dependencies**: None (can run in parallel)

#### Job 2: Silver Transformation
- **Name**: `Job_02_Silver_Transformation`
- **Tasks**: 9 tasks (one per silver notebook)
- **Dependencies**: None within job

#### Job 3: Gold Dimensions
- **Name**: `Job_03_Gold_Dimensions`
- **Tasks**: 5 tasks (dimension notebooks)
- **Dependencies**: None within job (can run in parallel)

#### Job 4: Gold Facts ⚠️ DEPENDS ON JOB 3
- **Name**: `Job_04_Gold_Facts`
- **Tasks**: 4 tasks with internal dependencies:
  - `task_fact_orders` (runs first)
  - `task_fact_order_items` (depends on fact_orders)
  - `task_fact_payments` (depends on fact_orders)
  - `task_fact_reviews` (depends on fact_orders)

**Task Dependency Configuration**:
```json
{
  "tasks": [
    {
      "task_key": "task_fact_orders",
      "notebook_task": {"notebook_path": ".../nb_fact_orders"}
    },
    {
      "task_key": "task_fact_order_items",
      "depends_on": [{"task_key": "task_fact_orders"}],
      "notebook_task": {"notebook_path": ".../nb_fact_order_items"}
    },
    {
      "task_key": "task_fact_payments",
      "depends_on": [{"task_key": "task_fact_orders"}],
      "notebook_task": {"notebook_path": ".../nb_fact_payments"}
    },
    {
      "task_key": "task_fact_reviews",
      "depends_on": [{"task_key": "task_fact_orders"}],
      "notebook_task": {"notebook_path": ".../nb_fact_reviews"}
    }
  ]
}
```

#### Job 5: KPI Calculation
- **Name**: `Job_05_KPI_Calculation`
- **Tasks**: 9 tasks (KPI notebooks)
- **Dependencies**: None within job (can run in parallel)

### How to Create Jobs in Databricks UI

1. Navigate to **Workflows** → **Jobs** → **Create Job**
2. Enter job name (e.g., `Job_01_Bronze_Ingestion`)
3. Click **+ Add task** for each notebook
4. Configure task:
   - Task name: `task_ingest_orders`
   - Type: **Notebook**
   - Notebook path: Select from workspace
   - Cluster: Choose existing or create new
5. Set task dependencies (click task → "Depends on")
6. Save and run

---

## 8. Pipeline Orchestration

### Master Workflow: End-to-End Pipeline

**Name**: `Pipeline_ECommerce_End_to_End`

### Job Dependency Chain

```
┌─────────────────────┐
│ Job 1: Bronze       │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Job 2: Silver       │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Job 3: Gold Dims    │ ← MUST complete first
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Job 4: Gold Facts   │ ← Depends on Dimensions
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Job 5: KPIs         │
└─────────────────────┘
```

### Creating Master Workflow

1. Create new Job: **Workflows** → **Jobs** → **Create Job**
2. Name: `Pipeline_ECommerce_End_to_End`
3. Add **Job Tasks** (not notebook tasks):

**Task 1: Bronze**
- Task key: `run_bronze_job`
- Type: **Run Job**
- Job: `Job_01_Bronze_Ingestion`

**Task 2: Silver**
- Task key: `run_silver_job`
- Type: **Run Job**
- Job: `Job_02_Silver_Transformation`
- **Depends on**: `run_bronze_job`

**Task 3: Gold Dimensions**
- Task key: `run_gold_dims_job`
- Type: **Run Job**
- Job: `Job_03_Gold_Dimensions`
- **Depends on**: `run_silver_job`

**Task 4: Gold Facts** ⚠️ **CRITICAL DEPENDENCY**
- Task key: `run_gold_facts_job`
- Type: **Run Job**
- Job: `Job_04_Gold_Facts`
- **Depends on**: `run_gold_dims_job` ← **REQUIRED!**

**Task 5: KPIs**
- Task key: `run_kpi_job`
- Type: **Run Job**
- Job: `Job_05_KPI_Calculation`
- **Depends on**: `run_gold_facts_job`

### Schedule Configuration
- **Daily**: `0 0 2 * * ?` (2:00 AM daily)
- **Weekly**: `0 0 2 ? * MON` (Monday 2:00 AM)

### Monitoring & Alerts
Configure email notifications for:
- On Failure: Send to data team
- On Success: Optional

---

## 9. Implementation Steps

### Phase 1: Environment Setup (15 min)

**Create Catalog & Schemas**:
```sql
CREATE CATALOG IF NOT EXISTS ecommerce_orders;
USE CATALOG ecommerce_orders;

CREATE SCHEMA IF NOT EXISTS 01_bronze;
CREATE SCHEMA IF NOT EXISTS 02_silver;
CREATE SCHEMA IF NOT EXISTS 03_gold;
```

**Verify S3 Access**:
```python
df = spark.read.csv("s3://ecommerce-order-s3-bucket-.../olist_orders_dataset.csv", header=True)
df.show(5)
```

### Phase 2: Bronze Layer (30 min)
1. Run all 9 bronze notebooks manually
2. Verify table creation: `SHOW TABLES IN ecommerce_orders.01_bronze`
3. Create `Job_01_Bronze_Ingestion`
4. Test job execution

### Phase 3: Silver Layer (30 min)
1. Run all 9 silver notebooks manually
2. Verify transformations
3. Create `Job_02_Silver_Transformation`
4. Test job execution

### Phase 4: Gold Layer (45 min)
1. **Run dimensions FIRST** (5 notebooks)
2. Verify dimensions: `SELECT COUNT(*) FROM ecommerce_orders.03_gold.dim_customer`
3. **Run facts AFTER dimensions** (4 notebooks)
4. Create `Job_03_Gold_Dimensions` (5 tasks)
5. Create `Job_04_Gold_Facts` (4 tasks with dependencies)
6. **Ensure Job 4 cannot run until Job 3 completes**

### Phase 5: KPI Layer (20 min)
1. Run all 9 KPI notebooks
2. Verify KPI calculations
3. Create `Job_05_KPI_Calculation`

### Phase 6: Orchestration (15 min)
1. Create master workflow `Pipeline_ECommerce_End_to_End`
2. Link all 5 jobs with dependencies
3. Test end-to-end run
4. Set up schedule and alerts

---

## 10. Testing & Validation

### Bronze Validation
```sql
-- Check row counts
SELECT COUNT(*) FROM ecommerce_orders.01_bronze.orders;

-- Check for duplicates
SELECT order_id, COUNT(*) 
FROM ecommerce_orders.01_bronze.orders 
GROUP BY order_id 
HAVING COUNT(*) > 1;
```

### Silver Validation
```sql
-- Verify data types
DESCRIBE TABLE ecommerce_orders.02_silver.orders;

-- Check date ranges
SELECT 
    MIN(order_purchase_timestamp) AS earliest,
    MAX(order_purchase_timestamp) AS latest
FROM ecommerce_orders.02_silver.orders;
```

### Gold Validation
```sql
-- Check dimension cardinality
SELECT COUNT(DISTINCT customer_id) FROM ecommerce_orders.03_gold.dim_customer;

-- Verify fact-dimension joins (no orphans)
SELECT COUNT(*) AS orphaned_orders
FROM ecommerce_orders.03_gold.fact_orders f
LEFT JOIN ecommerce_orders.03_gold.dim_customer d 
    ON f.customer_id = d.customer_id
WHERE d.customer_id IS NULL;
-- Should return 0
```

### KPI Validation
```sql
SELECT * FROM ecommerce_orders.03_gold.kpi_aov;
SELECT * FROM ecommerce_orders.03_gold.kpi_revenue_by_month ORDER BY month;
```

---

## Troubleshooting Guide

| Issue | Cause | Solution |
|-------|-------|----------|
| Bronze job fails | S3 access denied | Check IAM roles and bucket permissions |
| Silver job fails | Schema mismatch | Enable `.option("mergeSchema", "true")` |
| Fact table empty | Dimensions not created | Verify Gold Dims job completed first |
| KPI shows NULL | Missing joins | Check fact table population |
| Workflow stuck | Dependency deadlock | Review task dependencies |

---

## Summary Checklist

### ✅ Bronze Layer
- [ ] Created 9 bronze tables
- [ ] Verified raw data loaded from S3
- [ ] Created Bronze Job with 9 tasks

### ✅ Silver Layer
- [ ] Created 9 silver tables
- [ ] Validated data transformations
- [ ] Created Silver Job with 9 tasks

### ✅ Gold Layer
- [ ] Created 5 dimension tables **FIRST**
- [ ] Created 4 fact tables **AFTER** dimensions
- [ ] Verified fact-dimension relationships
- [ ] Created 2 Gold Jobs (Dims + Facts)
- [ ] **Ensured Facts Job depends on Dims Job**

### ✅ KPI Layer
- [ ] Created 9 KPI tables
- [ ] Validated KPI calculations
- [ ] Created KPI Job with 9 tasks

### ✅ Orchestration
- [ ] Created master workflow
- [ ] Set dependencies: Bronze → Silver → Dims → Facts → KPIs
- [ ] Configured scheduling
- [ ] Set up alerts
- [ ] Tested end-to-end execution

---

## Naming Conventions

### Notebooks
```
Bronze:  nb_ingest_<table_name>
Silver:  nb_transform_<table_name>
Gold:    nb_dim_<dimension_name> or nb_fact_<fact_name>
KPIs:    nb_kpi_<metric_name>
```

### Tables
```
Bronze:  ecommerce_orders.01_bronze.<table_name>
Silver:  ecommerce_orders.02_silver.<table_name>
Gold:    ecommerce_orders.03_gold.dim_<name>
         ecommerce_orders.03_gold.fact_<name>
         ecommerce_orders.03_gold.kpi_<name>
```

### Jobs
```
Job_01_Bronze_Ingestion
Job_02_Silver_Transformation
Job_03_Gold_Dimensions
Job_04_Gold_Facts
Job_05_KPI_Calculation
Pipeline_ECommerce_End_to_End
```

---

**Documentation Version**: 1.0  
**Last Updated**: March 2026  
**Author**: Data Engineering Team
