# E-Commerce Lakehouse — Databricks Medallion Architecture
 
A production-style Lakehouse built on **Databricks** and **Delta Lake** that ingests raw e-commerce data from two disconnected source systems, transforms it through a three-layer **Medallion Architecture**, and delivers a clean, analytics-ready star schema stored in **Unity Catalog**.
 
The project demonstrates how a Lakehouse unifies data engineering and analytics workloads on a single platform — the same Delta Tables that are written by PySpark pipelines are directly queryable for reporting without any export or transfer step.
 
---
 
## Table of Contents
 
- [Author](#author)
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Data Sources](#data-sources)
- [Data Flow](#data-flow)
- [Medallion Layers](#medallion-layers)
  - [Bronze Layer](#bronze-layer)
  - [Silver Layer](#silver-layer)
  - [Gold Layer](#gold-layer)
- [Orchestration](#orchestration)
- [Data Model](#data-model)
- [Data Quality Issues Resolved](#data-quality-issues-resolved)
- [Tech Stack](#tech-stack)
- [Naming Conventions](#naming-conventions)
- [Project Structure](#project-structure)
- [Future Work](#future-work)
 
---
 
## Author
 
**Mohamed Taha Abo Heiba** — Junior Data Engineer
 
I built this project to apply what I have learned in data engineering — from working with raw, messy data all the way to producing business-ready models. It demonstrates an end-to-end data pipeline covering data ingestion, transformation, and dimensional modeling, following industry best practices including Medallion Architecture, ETL process design, and star schema data modeling.
 
I put a lot of effort into writing clean, well-organized code with a clear structure and comments throughout each notebook. I hope you find the project useful and easy to follow.
 
- GitHub: [github.com/MoTahaAboHeiba](https://github.com/MoTahaAboHeiba)
- LinkedIn: [linkedin.com/in/mohamed-taha-abo-heiba](https://linkedin.com/in/mohamed-taha-abo-heiba)
 
---
 
## Project Overview
 
Raw e-commerce data arrives from two disconnected systems — a CRM and an ERP — across 6 CSV files. The data contains encoding inconsistencies, invalid dates, corrupted price values, non-standardized categorical codes, and duplicate customer records across sources.
 
This project builds a Lakehouse on Databricks that processes that data through three layers:
 
- **Bronze** — raw ingestion, zero transformation, full fidelity preservation
- **Silver** — cleaning, conforming, and standardizing using PySpark
- **Gold** — dimensional modeling using Spark SQL, producing a star schema ready for analytics
 
All layers are stored as **Delta Tables**, which provides ACID transaction guarantees, schema enforcement, and time travel on every table in the Lakehouse. The full pipeline runs end-to-end through a scheduled **Databricks Job** with a three-task dependency graph.
 
---
 
## Architecture
 
```
CRM Sources (3 CSV)        ERP Sources (3 CSV)
        |                          |
        v                          v
   [ Bronze Layer ]  <--  Raw ingestion, no transformation
        |
        v
   [ Silver Layer ]  <--  PySpark: cleaning, conforming, standardizing
        |
        v
   [ Gold Layer ]    <--  Spark SQL: star schema, business-ready models
        |
        v
   Delta Tables in Unity Catalog
```
 
The Lakehouse pattern means there is no separation between the storage layer and the compute layer. PySpark writes Delta Tables, and those same tables are immediately available for SQL analytics — no data movement, no separate warehouse load.
 
---
 
## Data Sources
 
| Source | File | Description |
|--------|------|-------------|
| CRM | `cust_info.csv` | Customer master data |
| CRM | `prd_info.csv` | Product catalog |
| CRM | `sales_details.csv` | Sales transactions |
| ERP | `CUST_AZ12.csv` | Customer demographics |
| ERP | `LOC_A101.csv` | Customer location data |
| ERP | `PX_CAT_G1V2.csv` | Product category hierarchy |
 
---
 
## Data Flow
 
```
Sources          Bronze Layer           Silver Layer            Gold Layer
-------          ------------           ------------            ----------
CRM ─────────► crm_cust_info    ────► customer_info    ───┐
               crm_prd_info     ────► product_info     ───┼──► Dim_Customer
               crm_sales_detail ────► Sales_Details    ───┤
ERP ─────────► erp_cust_az12   ────► erp_customer_info ───┼──► Dim_Product
               erp_loc_a101    ────► Customer_location ───┤
               erp_px_cat_g1v2 ────► erp_categories   ───┴──► Fact_Sales
```
 
---
 
## Medallion Layers
 
### Bronze Layer
 
Reads all 6 CSV files from source and writes them directly into Delta Tables with no transformation applied. Column names and data types are preserved exactly as received. The Bronze layer is the immutable source of truth — if anything goes wrong downstream, this layer is the recovery point.
 
**Tables:**
- `bronze_layer.crm_cust_info`
- `bronze_layer.crm_prd_info`
- `bronze_layer.crm_sales_details`
- `bronze_layer.erp_cust_az12`
- `bronze_layer.erp_loc_a101`
- `bronze_layer.erp_px_cat_g1v2`
 
---
 
### Silver Layer
 
Reads from Bronze Delta Tables and applies data quality rules, cleaning, and standardization using PySpark. All columns are renamed to consistent, business-readable names. Each source system gets its own notebook with transformations specific to the quality issues in that source.
 
**Tables:**
- `silver_layer.customer_info`
- `silver_layer.product_info`
- `silver_layer.Sales_Details`
- `silver_layer.erp_customer_info`
- `silver_layer.Customer_location`
- `silver_layer.erp_categories_details`
 
---
 
### Gold Layer
 
Reads from Silver Delta Tables and builds dimensional models using Spark SQL. CRM and ERP sources are integrated at this layer — customer data from three separate Silver tables is joined and conformed into a single `Dim_Customer`. The output is a clean star schema optimized for analytical queries.
 
**Tables:**
- `gold_layer.Dim_Customer`
- `gold_layer.Dim_Product`
- `gold_layer.Fact_Sales`
 
---
 
## Orchestration
 
The pipeline is orchestrated using **Databricks Jobs** with a three-task dependency graph enforcing layer execution order:
 
```
Bronze_Layer  ──►  Silver_Layer  ──►  Gold_Layer
   (27s)             (5m 54s)          (10m 57s)
```
 
Each task runs on a **Serverless** cluster. Dependencies are enforced at the Job level — Bronze must succeed before Silver starts, Silver must succeed before Gold starts. A failure at any task stops the pipeline immediately.
 
The orchestration notebooks include:
- Timestamped logging per notebook execution
- Execution duration tracking
- Hard failure propagation — exceptions are re-raised, not swallowed
- 30-minute timeout per individual notebook
 
The pipeline DAG screenshot is available in the [`Docs/ETL_Pipeline_DAG.jpeg`](./Docs/ETL_Pipeline_DAG.jpeg) folder.
 
---
 
## Data Model
 
Star schema with two-dimensional tables and one fact table:
 
```
Dim_Product ──────────────────── Fact_Sales ──────────────────── Dim_Customer
  Product_Key (PK)                 Order_Number                    Customer_Key (PK)
  Product_Id                       Product_Key (FK)                Customer_Id
  Product_Number                   Customer_Key (FK)               Customer_Number
  Product_Name                     Order_Date                      First_Name
  Category_Id                      Ship_Date                       Last_Name
  Category                         Due_Date                        Country
  SubCategory                      Price                           Marital_Status
  Maintenance                      Quantity                        Gender
  Cost                             Sales_Amount                    Birth_Date
  Product_Line                                                      Create_Date
  Start_Date
 
                              Sales_Amount = Price * Quantity
```
 
All architecture diagrams are available in the [`Docs/ETL_Pipeline_DAG.jpeg`](./Docs/ETL_Pipeline_DAG.jpeg) folder:
 
- `Data_model.jpg` — Star schema design
- `Date_Lineage_Diagram.jpg` — Data flow across all three layers
- `Data_Integration.jpg` — Source system integration map
 
Diagrams were designed using draw.io.
 
---
 
## Data Quality Issues Resolved
 
### CRM — Customer Info
- Removed records with null `customer_id`
- Trimmed whitespace from all string columns
- Decoded gender codes: `M` to `Male`, `F` to `Female`, anything else to `Unknown`
- Decoded marital status codes: `S` to `Single`, `M` to `Married`, anything else to `Unknown`
 
### CRM — Product Info
- Replaced null product costs with `0`
- Fixed date format inconsistency — parsed `M/d/yyyy` string to proper `DateType`
- Decoded product line codes: `M` to `Mountain`, `R` to `Road`, `S` to `Other Sales`, `T` to `Touring`
- Extracted `category_id` embedded inside the `product_number` field and cleaned the remainder
 
### CRM — Sales Details
- Dates were stored as 8-digit integers in `yyyyMMdd` format — cast to string then parsed to `DateType`
- Zero values and malformed dates set to null
- Null and negative prices recalculated as `sales_amount / quantity` where quantity was non-zero
 
### ERP — Customer Demographics
- Stripped `NAS` prefix from customer ID values
- Removed records where `birthdate` is a future date
- Normalized gender: `M` to `Male`, `F` to `Female`, null or empty to `Unknown`
 
### ERP — Customer Location
- Removed dashes from customer number to match CRM format for joining
- Normalized country codes: `US` and `USA` to `United States`, `DE` to `Germany`, empty to `Unknown`
 
### ERP — Product Categories
- Converted `maintenance` flag from `YES`/`NO` string to proper Boolean
 
---
 
## Tech Stack
 
| Component | Technology |
|-----------|-----------|
| Paradigm | Lakehouse Architecture |
| Platform | Databricks Community Edition |
| Storage Format | Delta Lake (Parquet + Transaction Log) |
| Catalog | Unity Catalog |
| Bronze and Silver | PySpark |
| Gold | Spark SQL |
| Orchestration | Databricks Jobs (3-task DAG) |
| Compute | Serverless Cluster |
| Diagrams | draw.io |
| Version Control | Git / GitHub |
 
---
 
## Naming Conventions
 
| Layer | Pattern | Example |
|-------|---------|---------|
| Bronze | `<source>_<entity>` | `crm_cust_info` |
| Silver | `<source>_<entity>` | `crm_customer_info` |
| Gold Dimensions | `dim_<entity>` | `dim_customers` |
| Gold Facts | `fact_<entity>` | `fact_sales` |
| Surrogate Keys | `<table>_key` | `customer_key` |
| Technical Columns | `dwh_<column_name>` | `dwh_load_date` |
 
Full details in [Naming_Conventions.md](./docs/Naming_Conventions.md)
 
---
 
## Project Structure
 
```
E-Commerce-Databricks-Project/
    Scripts/
        Bronze Layer/
            Bronze
        Silver Layer/
            Silver_CRM_Customer_info
            Silver_CRM_Product_info
            Silver_CRM_Sales_Details
            Silver_ERP_Category
            Silver_ERP_customer
            Silver_ERP_location
        Gold Layer/
            Gold_Dim_customers
            Gold_Dim_products
            Gold_Fact_Sales
        Orchestration/
            notebook_runner
            silver_orchestration
            gold_orchestration
    docs/
        Data_model.jpg
        Date_Lineage_Diagram.jpg
        Data_Integration.jpg
        ETL_Pipeline_DAG.png
        Naming_Conventions.md
```
 
---
 
## Future Work
 
- Build analytical queries and reporting views on top of the Gold layer
- Connect Gold Delta Tables to Power BI or Databricks SQL for visualization
- Replace full overwrite with incremental load logic in Bronze and Silver
- Add a data validation layer between Silver and Gold to enforce row counts and null thresholds
- Add `dwh_load_date` technical column to all Silver and Gold tables for full auditability
