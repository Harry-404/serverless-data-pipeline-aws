# serverless-data-pipeline-aws

**Deploy and analyze an e-commerce Orders table in DynamoDB using an end-to-end serverless AWS data pipeline. Full step-by-step creation: DynamoDB, AWS Glue (Crawler + ETL), S3 Parquet Data Lake, Athena SQL analytics, and cleanup.**

***

## Architecture Overview

- **Source:** DynamoDB Table (`Orders`) with sample orders data.
- **AWS Glue Crawler:** Catalogs the DynamoDB table to Glue Data Catalog.
- **Glue ETL Job:** Filters and transforms Order data to Parquet, stores in S3.
- **S3 Glue Crawler:** Catalogs Parquet data lake for Athena.
- **Amazon Athena:** SQL analytics directly on S3.
- **Optional:** Visualization with QuickSight.
- **IAM:** Separate roles for each pipeline step (principle of least privilege).

***

## Prerequisites

- AWS CLI/configured console access.
- Permissions for DynamoDB, S3, Glue, Athena.
- Sample data file: `sample-data/orders.json` (see below).

***

## Step-by-Step Lab

### 1. Create DynamoDB Table & Insert Data

- Go to **DynamoDB Console → Tables → Create Table**
- Name: `Orders`
- Partition Key: `OrderID` (String)
- Sort Key: `OrderDate` (String)
- Set Capacity to On-Demand.

**Sample Orders:**

```json
{
  "OrderID": "01001",
  "OrderDate": "2025-09-05",
  "Customer": "Alice",
  "Amount": 250,
  "Status": "Shipped"
}
```

```json
{
  "OrderID": "01002",
  "OrderDate": "2025-09-05",
  "Customer": "Bob",
  "Amount": 400,
  "Status": "Pending"
}
```

*Add 10–15 rows total for meaningful analytics.*

***

### 2. Catalog DynamoDB with Glue Crawler

- **AWS Glue Console → Crawlers → Add Crawler**
- Name: `OrdersCrawler`
- Source: DynamoDB > Select `Orders` table
- IAM Role: `AWSGlueServiceRole-Orders` (DynamoDB ReadOnly + Glue minimal)
- Target Database: `ordersdb` (lowercase)
- Run crawler to infer schema (OrderID, OrderDate, Customer, Amount, Status).

***

### 3. Glue ETL Job — Filter & Export to S3 Parquet

- **Glue Studio: Create visual job OR use PySpark script below.**
- Source: Glue Catalog table `ordersdb.orders`

**PySpark ETL Script Example:**

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

datasource = glueContext.create_dynamic_frame.from_catalog(
    database="ordersdb",
    table_name="orders"
)
filtered = Filter.apply(datasource, lambda x: x["status"] == "Shipped")
glueContext.write_dynamic_frame.from_options(
    frame=filtered,
    connection_type="s3",
    connection_options={"path": "s3://my-orders-analytics/shipped/"},
    format="parquet"
)
job.commit()
```

- Target: **Amazon S3 Folder:** `s3://my-orders-analytics/shipped/`
- Format: Parquet, Compression: Snappy (recommended)

***

### 4. Catalog Cleaned S3 Data (S3 Crawler)

- **Glue Console → Crawlers → Add Crawler**
- Name: `OrdersS3Crawler`
- Source: S3 bucket/folder: `s3://my-orders-analytics/shipped/`
- Role: (e.g., `S3buck`, with S3 full, Glue, DynamoDB read-only)
- Target Database: `ordersdb`
- Run crawler to catalog Parquet data

***

### 5. Athena SQL Analytics

- **Athena Console:**
  - Query result location: `s3://athena-query-results-54/`
  - Data source: AwsDataCatalog, Database: `ordersdb`, Table: `shipped`

**Sample Athena Queries:**

```sql
-- Top customers by spend
SELECT Customer, SUM(Amount) AS TotalSpent
FROM shipped
WHERE Status = 'Shipped'
GROUP BY Customer
ORDER BY TotalSpent DESC;

-- Monthly Revenue trend
SELECT OrderDate, SUM(Amount) AS MonthlyRevenue
FROM shipped
WHERE Status = 'Shipped'
GROUP BY OrderDate;
```

***

### 6. Visualization (Optional)

- Connect Amazon QuickSight to Athena
- Example Dashboards:
  - Customer vs. TotalSpent (bar chart)
  - Monthly revenue trend (line chart)

***

## Cleanup

- **DynamoDB:** Delete the `Orders` table.
- **AWS Glue:** Delete all ETL jobs, crawlers, and databases (delete tables first, then database).
- **Amazon S3:** Empty and delete `my-orders-analytics` and Athena output buckets.
- **Athena:** Drop all Athena tables.
- **Do not delete** buckets prefixed with `aws-glue-assets-*` (internal use).

***

## Best Practices

- Use IAM least privilege for all roles.
- Use Parquet for cost/performance.
- Partition S3 data by date/customer for scalability.
- Clean Data Catalog and S3 buckets regularly.

***

## Connect
<a href="https://www.linkedin.com/in/hiranmaya-biswas-505a1823a/" target="_blank"> <img src="https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin" alt="LinkedIn" height="30"> </a> <a href="https://github.com/Harry-404" target="_blank" style="margin-left:10px;"> <img src="https://img.shields.io/badge/GitHub-Follow-black?logo=github" alt="GitHub" height="30"> </a>
