# 🚚 Supply Chain End-to-End ETL Pipeline

![Azure](https://img.shields.io/badge/Cloud-Azure-blue) ![ADF](https://img.shields.io/badge/Orchestration-Data%20Factory-informational) ![Databricks](https://img.shields.io/badge/Compute-Databricks-orange) ![Status](https://img.shields.io/badge/Pipeline-Production%20Ready-success)

## 📋 Executive Summary
**Problem:** A logistics company struggled with fragmented inventory data and rising cloud costs due to redundant full-load data processing every 24 hours. The lack of a centralized data warehouse made supplier performance tracking impossible.

**Solution:** Designed a dynamic, metadata-driven ETL pipeline using **Azure Data Factory** and **Databricks**. The system utilizes a **Control Table** architecture to dynamically manage ingestion for multiple tables and implements **Watermark-based Incremental Loading** to process only changed data.

**Outcome:**
* **Cost Optimization:** Reduced compute costs by **~40%** by processing only deltas (incremental changes) instead of full historical loads.
* **Scalability:** The Control Table design allows adding new source tables without modifying the pipeline code.
* **Data Quality:** Implemented a Medallion Architecture (Bronze/Silver/Gold) to ensure 100% data consistency for the "Supplier Ranking" dashboard.

---

## 🏗️ Architecture
The pipeline follows a modern **Lakehouse Architecture** leveraging the Medallion design pattern.

![Architecture Diagram](https://github.com/smitpatel8/supply-chain-etl/tree/main/images/diagram1.png)

### 🛠️ Tech Stack
* **Orchestration:** Azure Data Factory (ADF)
* **Processing:** Azure Databricks (PySpark)
* **Storage:** Azure Data Lake Gen 2 (ADLS) & SQL Server
* **Security:** Azure Key Vault (Secret Management)
* **Metadata Management:** SQL Control Table

---

## ⚙️ Engineering Logic: Metadata-Driven Ingestion
*Unlike standard hard-coded pipelines, this project uses a **Dynamic Control Table** to manage ingestion state.*

### The Control Table Strategy
There is a single SQL table (`dbo.ControlTable`) acting as the brain of the pipeline. It stores metadata: `Source_Table`, `Load_Type` (Full/Incremental), and `Watermark_Value`.

![Control Table Logic](https://github.com/smitpatel8/supply-chain-etl/tree/main/images/diagram2.png)

### The Pipeline Flow (ADF)
1.  **Lookup Control Table:** Fetches list of tables where `Active_Indicator = 1`.
2.  **Dynamic Iteration:** A `ForEach` loop iterates through every table in the list.
3.  **Load Logic Router:**
    * **IF Incremental:** * Fetches `MAX(Last_Modified_Date)` from source.
        * Copies only rows where `SourceDate > LastWatermark`.
        * **Stored Procedure:** Updates the Control Table with the new High Watermark.
    * **IF Full Load:** * Truncates destination and reloads all data.
        * **Stored Procedure:** Updates the flag to switch to "Incremental" for the next run.

---

## 🔄 Data Transformation (Medallion Architecture)
[View the complete PySpark Notebooks here](https://github.com/smitpatel8/Supply-Chain-ETL/tree/main/pyspark_notebooks)

The raw data moves through three quality layers using **Azure Databricks**:

#### 1. 🥉 Bronze Layer (Raw Ingestion)
* **Goal:** Land data "as-is" from ADF.
* **Checks:** Validated schema types and handled initial `NULL` values.
* **Optimization:** Saved as **Parquet** (Snappy compressed) for storage efficiency.

#### 2. 🥈 Silver Layer (Clean & Enriched)
* **Goal:** Apply business rules and clean data.
* **Logic Applied:**
    * **Supplier Negotiation:** Categorized scores into 'High', 'Medium', 'Low'.
    * **Transportation:** Normalized shipping modes (Air vs Ground).
    * **Financials:** Calculated `Total_Cost` (Quantity * Unit Price + Shipping).

#### 3. 🥇 Gold Layer (Aggregated for BI)
* **Goal:** Business-ready tables for reporting.
* **Logic Applied:**
    * **Supplier Ranking:** Ranked suppliers based on defect rates and delivery speed.
    * **Recommendation Engine:** Generated a 'Preferred Supplier' list for procurement teams.

---

## 💡 Key Design Decisions

| Decision | Why I chose this? |
| :--- | :--- |
| **Control Table vs. Hardcoding** | Hardcoding 10 pipelines for 10 tables is unmaintainable. A Control Table allows me to onboard new datasets just by adding a row in SQL, zero code changes required. |
| **Watermarking (Incremental)** | Processing the full history daily wastes compute credits. Watermarking ensures we only pay to process the ~5% of data that actually changed. |
| **Azure Key Vault** | I utilized Key Vault to decouple credentials from the ADF code, preventing security leaks in the GitHub repository. |

---

## 💰 Cost Analysis (Estimated)
*Optimized for a mid-sized workload (10GB daily volume).*

| Service | Configuration | Est. Monthly Cost |
| :--- | :--- | :--- |
| **Azure Data Factory** | Pay-per-activity (1 run/day) | ~$20.00 |
| **Databricks** | Standard Cluster (DS3_v2), 1 hr/day | ~$45.00 |
| **Data Lake Gen2** | Hot Tier (Storage + Transactions) | ~$5.00 |
| **Total** | | **~$70.00 / month** |

---

## 🚀 Pipeline Execution Result
*Screenshot demonstrating the Stored Procedure successfully updating the Watermark after an incremental load:*

![Pipeline Run](https://github.com/smitpatel8/supply-chain-etl/tree/main/images/diagram4.png)

## 📂 Source Data
* [Download the Datasets](https://github.com/smitpatel8/Supply-Chain-ETL/tree/main/Datasets)
