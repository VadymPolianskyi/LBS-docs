# Development and Data Modeling Standards
This document defines best practices for development and data modeling across these layers, ensuring consistency, scalability, and governance. It establishes:
- Table and column naming conventions.
- Metadata governance and lineage tracking.
- Dimensional modeling techniques by [Kimball](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/)
-  Implementation of SCD Type 2 for historical data.

## Bronze 
The Bronze Layer serves as the ingestion point for raw data from structured sources, flat files, and landing zone. This layer retains data in its original format with minimal transformations (no data/schema updates, only appends), ensuring data lineage and traceability for downstream processing.
API data follows `landing` → `bronze` transformation process.

#### Data Structure & Organization

- All data is stored in **OneLake** under `/bronze/{source_system}/` folders.
- Folder structure mirrors API endpoints or source tables.
- Tables are absolutely going to be named just as in source.
- Landing data is ephemeral - once metadata is captured, data moves to the final Bronze storage. 
-   Examples:
    ```
    /landing/mysite__com/posts/69/comments/data.json  # Temporary Landing Zone
    /bronze/mysite__com/posts/69/comments/data.json   # Final Bronze storage
    ```
#### Naming Conventions for Structured Tables
Names follow their **original usage**, preserving formatting and conventions (e.g., *[ad_clickthrough]* instead of *[ad click through],* maintaining original compound structure). This ensures consistency with source data and established naming patterns.

Structured tables in the **Bronze Layer** are stored in **Delta format**, which has specific constraints on column naming. Delta does not allow:

- Spaces in column names.    
- Special characters such as `,`, `/`, `()`, `-`.  
- Leading numeric values in column names.

To comply with Delta format constraints, column names should be transformed using the following rules:
1.  Replace spaces with underscores (`**_**`)
    -   Example: `Customer Name` → `customer_name`
2.  Remove special characters and use underscores
    -   Example: `Order-Date` → `order_date`
    -   Example: `Revenue ($)` → `revenue_usd`
3.  Ensure column names start with a letter
    -   Example: `2023_sales` → `sales_2023`
4.  Convert all column names to lowercase to maintain consistency
    -   Example: `UserEmail` → `user_email`
    
By enforcing these rules, the Bronze Layer maintains compatibility with Delta Lake storage while ensuring consistency and readability for downstream processes.

#### Incremental Load
Azure Data Factory (ADF) is responsible for orchestrating incremental data ingestion across multiple sources. The ingestion pipeline is designed to handle different types of data:

- **API Data**:
	- Extraction is managed by an Azure Function, which fetches API responses and inserts them into the Landing Zone in JSON format.
	- After this the Spark job transforms and moves the data into the Bronze Layer.
	- As the last step, another Azure Function stores the last successful API request timestamp in a watermark table (Redis).
 	- The last step is to update the last successful timestamp in the watermark table. 

- **Files**:
	- ADF ingests files directly into Bronze, using file metadata (LastModifiedDate) or partitioned folder structures to track new and modified files.
	- Utilize ADF’s `filtering by LastModifiedDate` to process only files updated since the last ingestion.
    - Organize files in a date-based partitioned structure (`/year/month/day/`) for efficient ingestion.

- **Structured Data (Tables)**:
	- The Spark job loads structured data incrementally. Since source tables lack update timestamps, a high watermark approach using control tables is used to track processed records.

Incremental ingestion ensures that only new or changed records are processed, reducing resource consumption and improving efficiency.
	
#### Processing Guidelines
- Schema-on-read: Bronze layer does not enforce schemas; data is interpreted at later stages.
- Immutable storage: No updates or deletions occur in this layer.
- Partitioning: The ingestion runs once a day for each source. Data should be partitioned by ingestion date (`year/month/day`) **???** to optimize retrieval.

#### Metadata
- Administrative metadata is stored separately in `{source__system_name}__mtdt` subfolder. [More details](https://dev.azure.com/azurereleasemanager/londonedu-dataplatform/_wiki/wikis/londonedu-dataplatform.wiki/5535/Storage-and-Compute?anchor=bronze-administrative-metadata)
- Technical metadata for the data from structured sources is stored in Delta log files `_delta_log/xxxxxxx.json`.

## Silver
The Silver Layer refines raw data from the Bronze Layer, ensuring it is validated, cleansed, deduplicated, and structured for analytical use. 
- The structure of the table mirrors source structure
- Data format: **Delta** 
- Data from Bronze is processed to remove duplicates, standardize formats, and enforce schema consistency.
- Enforce business rules and apply quality checks before storing data.
- Maintains only the latest version of each record.

#### Naming Conventions
- Table Naming: `<original_source_table>` 
- Columns Naming: keeps the same as in source.
	- Column names with **lowercase** letters with **underscores** `"_"` as delimiters (e.g., `user_sessions`).
	- Spaces, special characters (`-`, `/`, `()`) are replaced with underscores (`_`).
	- In case of issues with `source format -> delta` mapping, check **Naming Conventions for Structured Tables** above    

#### Metadata
Additional metadata is introduced in the Silver Layer to enhance data traceability and governance:

- Transformation Lineage: Track how data is processed and transformed.
- Data Quality Metrics: Store validation results for completeness, consistency, and accuracy. **???**
- Processing Timestamps:
    -   `processing_start` – Timestamp when data was processed into Silver.
    -   `processing_end` – Timestamp of last modification.
- Source System Reference:
    -   `source_system` – Identifies the data origin.
    -   `pipeline_id` – Tracks the applied transformation pipeline.

#### Processing Guidelines
- Deduplication: Remove duplicate records based on business-defined unique keys.
- Schema Enforcement: Apply strict column definitions to prevent inconsistencies.
- Partitioning: Optimize query performance by partitioning large datasets on relevant columns (e.g., `dt_transaction`).
- Incremental Updates:
    -   Merge new and updated records efficiently.
    -   Maintain historical changes for auditability only in the Gold Layer.
-   Logging & Monitoring: Capture job execution details, failures, and validation errors.

## Gold
The Gold Layer is optimized for reporting, business intelligence, and analytical workloads. It structures data into fact and dimension tables, following Kimball’s Star Schema principles and implementing Slowly Changing Dimensions (SCD Type 2) for historical tracking.

### Naming Conventions

The purpose of these naming conventions is to:

- Enhance readability and understanding of data structures.
- Facilitate collaboration among team members by providing a common language.
- Simplify maintenance and updates by ensuring that all components are easily identifiable.
- Use a consistent format throughout the platform.

#### Basic Principles
1. **Clarity and Descriptiveness**:
    - Names should clearly indicate the purpose or content of the resource.
    - Avoid using obscure acronyms or abbreviations unless widely recognized.
    - **British English** is used consistently in all names, ensuring adherence to regional spelling conventions (e.g., *centre* instead of *center*, *analyse* instead of *analyze* and *metre* instead of *meter*). However, for raw data stored in the Bronze layer, the original version is retained. In the gold layer tables, all names are standardised to British English.
    - Column names with a **maximum of 5 terms** described avoiding abbreviation
    - Avoid using accents, spaces and other special characters, recommended only letters, numbers and underline/uderscore
    - Ensure standardization of **date/time** fields across datasets. Use `created_at`, `updated_at`, `event_date`, `event_timestamp` instead of ambiguous names like `date`, `time`.
 
#### Naming Rules

- Warehouse Naming: `gold_<domain>` (e.g., `gold_marketing`)
- Schema Naming: `<product>`/`<business_use_case>` (e.g., `brand_and_marketing`)
- Table Naming:
    - Fact Tables: `fct_<event>` (e.g., `fct_transactions`)
    - Dimension Tables: `dim_<entity>` (e.g., `dim_customer`)
- General Column Naming:
    - `pk_` – Surrogate primary keys (e.g., `pk_customer_id`)
    - `fk_` – Foreign keys linking facts to dimensions (e.g., `fk_product_id`)
    - `nm_` – Name fields (e.g., `nm_product`)
    - `num_` – Numeric values (e.g., `num_price`)
    - `dt_` – Date and timestamp values (e.g., `dt_effective_start`, `dt_effective_end`)
    - `txt_` – Text descriptions (e.g., `txt_remarks`)
    - `bool_` – Boolean flags (e.g., `bool_is_active`)
        
#### Business Metadata
[TBD]: Pureview

#### Star Schema
Star schema consist of 2 types of table - facts and dimensions. It's used for Golden layer to to optimize query performance and analytical processing.
![Fact and Dimension table example](https://www.montecarlodata.com/wp-content/uploads/2024/08/fact-and-dimension-tables-in-data-warehousing.png)
A **Fact table** contains the numeric measures produced by an operational measurement event in the real world and are linked to dimensions using foreign keys. 
- It should be named with the prefix `fct_{name}` (e.g., `fct_transactions`).
- Fact table always contains foreign keys for each of its associated dimensions, as well as  date/time stamps. Fact tables are the primary target of computations and dynamic aggregations arising from queries.
-   Aggregate functions (SUM, COUNT, MIN, MAX, AVG) handle `null` values correctly.
-   Foreign keys should never be `null`; instead, use a default row in the dimension table to represent unknown values.

A **Dimension table** contains descriptive attributes that provide context to fact table measures. 
- It should be named with the prefix `dim_{name}` (e.g., `dim_customer`).
- Stores non-numeric descriptive attributes (e.g., customer names, product categories).  
- Serves as a lookup for fact tables via foreign keys.
- Implements **SCD Type 2** to maintain historical changes.
- Uses surrogate primary keys (`pk_`) instead of natural keys for better performance and consistency.
    

#### SCD Type 2    
[Slowly Changing Dimension (SCD) Type 2](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/type-2/) is used to track historical changes in dimension tables. This allows analytics and reporting to reflect changes in attributes over time. Each change will result in a new row, rather than updating existing records.

Key Components:
-   `pk_` Surrogate Key – Unique identifier for each record version.
-   `dt_start_date` & `dt_end_date` – Define the validity period of a record.
-   `bool_is_active` – Indicates the most recent active version (`TRUE/FALSE`).

Example `Dim_Product`:

![Diagram showing the function and structure of OneLake.](https://learn.microsoft.com/en-us/training/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/media/2-slowly-changing-dimension.png) 

The following example shows how to handle the business key in a type 2 SCD for the  `Dim_Products`  table using T-SQL.
```sql
IF EXISTS (SELECT 1 FROM dim_products WHERE pk_source_key = @ProductID AND bool_is_active = 'True')
BEGIN
    -- Existing product record
    UPDATE dim_products
    SET dt_valid_to = GETDATE(), bool_is_active = 'False'
    WHERE pk_source_key = @ProductID 
        AND bool_is_active = 'True';
END
ELSE
BEGIN
    -- New product record
    INSERT INTO dim_products (pk_source_key, nm_product_name, dt_start_date, dt_end_date, bool_is_active)
    VALUES (@ProductID, @ProductName, GETDATE(), '9999-12-31', 'True');
END
```
---
1.  When a new record is inserted, it gets a new `pk_source_key` and `dt_start_date`.
2.  When a change occurs, the previous record’s `dt_end_date` is updated, and a new row is added with `bool_is_active = TRUE`.
3.  Queries always use `bool_is_active = TRUE` to retrieve the latest record.

## Notebook using
[TBD]
### Best practices
- Use commentary (md cells) 
	- For context
	- For easy navigation
- Use classes and functions
- Separate notebooks
- Use folders and subfolders (on the Workspace level).
Example:
```
.
└── data
| └── silver/
|   └── {source system name}/
|     └── {schema}/
|       └── {table}.ipynb
└── data-quality
  └── silver/
    └── {source system name}/
      └── {schema}/
	└── {table}.ipynb
```

### Parameters
Template notebooks should be parameterized to support flexible reuse:
-   Source and target locations (paths)
-   Date ranges or timestamps for incremental processing
-   Source system identifiers
-   Transformation rules and configurations

Example parameters in notebook cells:
```python
source_path = "/bronze/source_system/entity/"
target_path = "/silver/domain/entity/"
ingestion_date = "2024-04-01"
```
Example parameters in notebook cells:

### Expected Returns
-   Clearly defined status outputs (e.g., "success", "failure", or specific error messages)
-   Number of rows processed, inserted, or updated **???**
