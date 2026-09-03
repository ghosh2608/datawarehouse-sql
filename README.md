<![CDATA[# 🏗️ Data Warehouse & Analytics Project

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![SQL Server](https://img.shields.io/badge/SQL%20Server-Express-CC2927?logo=microsoftsqlserver&logoColor=white)](https://www.microsoft.com/en-us/sql-server/sql-server-downloads)
[![Architecture](https://img.shields.io/badge/Architecture-Medallion-gold)](docs/data_architecture.png)

A production-style **Data Warehouse** built on Microsoft SQL Server, implementing the **Medallion Architecture** (Bronze → Silver → Gold) to consolidate ERP and CRM data into a Star Schema optimized for analytics and BI reporting.

---

## 📐 Architecture

This project follows the **Medallion Architecture** pattern with three distinct layers:

```
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │   BRONZE    │     │   SILVER    │     │    GOLD     │
  │  (Raw Data) │────▶│ (Cleansed)  │────▶│ (Analytics) │
  │             │     │             │     │             │
  │ • BULK      │     │ • Dedup     │     │ • Star      │
  │   INSERT    │     │ • Normalize │     │   Schema    │
  │ • Full      │     │ • Validate  │     │ • Surrogate │
  │   Reload    │     │ • Enrich    │     │   Keys      │
  └─────────────┘     └─────────────┘     └─────────────┘
        ▲
        │
  ┌─────────────┐
  │ CSV Sources │
  │ (CRM + ERP) │
  └─────────────┘
```

| Layer | Purpose | Implementation |
|-------|---------|----------------|
| **Bronze** | Raw data ingestion — stores data as-is from source systems | `BULK INSERT` from CSV files, full truncate-and-reload |
| **Silver** | Cleansed, standardized, deduplicated, and enriched data | Stored procedure with business rules, type conversions, and null handling |
| **Gold** | Business-ready dimensional model for analytics | SQL Views implementing a Star Schema with surrogate keys |

---

## 📊 Data Model (Gold Layer — Star Schema)

```
              ┌──────────────────┐
              │  dim_customers   │
              ├──────────────────┤
              │ customer_key (PK)│
              │ customer_id      │
              │ first_name       │
              │ last_name        │
              │ country          │
              │ gender           │
              │ marital_status   │
              │ birthdate        │
              └────────┬─────────┘
                       │
                       │ FK
                       ▼
              ┌──────────────────┐
              │   fact_sales     │
              ├──────────────────┤
              │ order_number     │
              │ customer_key (FK)│◄─── dim_customers
              │ product_key  (FK)│◄─── dim_products
              │ order_date       │
              │ shipping_date    │
              │ due_date         │
              │ sales_amount     │
              │ quantity         │
              │ price            │
              └────────┬─────────┘
                       │
                       │ FK
                       ▼
              ┌──────────────────┐
              │  dim_products    │
              ├──────────────────┤
              │ product_key (PK) │
              │ product_number   │
              │ product_name     │
              │ category         │
              │ subcategory      │
              │ cost             │
              │ product_line     │
              │ maintenance      │
              └──────────────────┘
```

For full field descriptions and data types, see the [Data Catalog](docs/data_catalog.md).

---

## 📂 Project Structure

```
datawarehouse_project/
│
├── datasets/                          # Raw source data (CSV files)
│   ├── source_crm/                    # CRM system exports
│   │   ├── cust_info.csv              #   Customer demographics
│   │   ├── prd_info.csv               #   Product catalog
│   │   └── sales_details.csv          #   Sales transactions
│   └── source_erp/                    # ERP system exports
│       ├── CUST_AZ12.csv              #   Customer birthdates & gender
│       ├── LOC_A101.csv               #   Geographic locations
│       └── PX_CAT_G1V2.csv           #   Product categories
│
├── scripts/                           # SQL scripts for ETL pipeline
│   ├── init_database.sql              # Database & schema initialization
│   ├── bronze/
│   │   ├── ddl_bronze.sql             # Bronze table definitions (6 tables)
│   │   └── proc_load_bronze.sql       # Bulk ingestion stored procedure
│   ├── silver/
│   │   ├── ddl_silver.sql             # Silver table definitions (typed + audit cols)
│   │   └── proc_load_silver.sql       # Cleansing & transformation procedure
│   └── gold/
│       └── ddl_gold.sql               # Star Schema views (2 dims + 1 fact)
│
├── tests/                             # Data quality validation
│   ├── quality_checks_silver.sql      # Silver layer DQ checks
│   └── quality_checks_gold.sql        # Gold layer integrity checks
│
├── docs/                              # Documentation & diagrams
│   ├── data_catalog.md                # Gold layer data dictionary
│   ├── naming_conventions.md          # Naming standards across all layers
│   ├── data_architecture.png          # Architecture diagram
│   ├── data_model.png                 # Star schema diagram
│   ├── data_flow.png                  # Data flow diagram
│   ├── data_integration.png           # Integration diagram
│   ├── ETL.png                        # ETL techniques reference
│   └── *.drawio                       # Editable Draw.io source files
│
├── .gitignore
├── LICENSE                            # MIT License
└── README.md                          # This file
```

---

## 🚀 Getting Started

### Prerequisites

- **[SQL Server Express](https://www.microsoft.com/en-us/sql-server/sql-server-downloads)** — free, lightweight database engine
- **[SQL Server Management Studio (SSMS)](https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms)** — GUI for executing scripts
- **Git** — for version control

### Setup & Execution

Run the scripts in order within SSMS:

```sql
-- Step 1: Initialize the database and schemas
-- Execute: scripts/init_database.sql
-- Creates: DataWarehouse database + bronze, silver, gold schemas

-- Step 2: Create Bronze tables
-- Execute: scripts/bronze/ddl_bronze.sql

-- Step 3: Load raw data from CSV into Bronze
-- Execute: scripts/bronze/proc_load_bronze.sql
EXEC bronze.load_bronze;

-- Step 4: Create Silver tables
-- Execute: scripts/silver/ddl_silver.sql

-- Step 5: Transform and load into Silver
-- Execute: scripts/silver/proc_load_silver.sql
EXEC silver.load_silver;

-- Step 6: Create Gold views (Star Schema)
-- Execute: scripts/gold/ddl_gold.sql

-- Step 7 (Optional): Run quality checks
-- Execute: tests/quality_checks_silver.sql
-- Execute: tests/quality_checks_gold.sql
```

> **Note:** Update the file paths in `proc_load_bronze.sql` to match your local CSV file locations before running Step 3.

---

## 🔧 ETL Pipeline Details

### Bronze Layer — Raw Ingestion
- **Strategy:** Full truncate-and-reload
- **Method:** `BULK INSERT` from CSV files with `FIRSTROW = 2` (skips headers)
- **Tables:** 6 tables mirroring CRM and ERP source structures
- **Error Handling:** `TRY...CATCH` with per-table load timing

### Silver Layer — Cleansing & Transformation
Key transformations performed by `silver.load_silver`:

| Transformation | Description |
|----------------|-------------|
| **Deduplication** | `ROW_NUMBER()` partitioned by customer ID, keeping latest record |
| **String Sanitization** | `TRIM()` on name fields to remove whitespace |
| **Value Normalization** | Code-to-label mapping (e.g., `'S'` → `'Single'`, `'M'` → `'Mountain'`) |
| **Date Validation** | Rejects invalid/future dates, converts integer dates to `DATE` type |
| **Business Rule Enforcement** | Recalculates `sales_amount` when inconsistent with `quantity × price` |
| **Key Parsing** | Extracts category ID and product key from composite keys |
| **Country Standardization** | Maps codes (`'DE'`, `'US'`, `'USA'`) to full country names |
| **Gender Harmonization** | Normalizes across CRM and ERP sources with fallback logic |

### Gold Layer — Dimensional Model
- **`dim_customers`** — Enriched customer dimension joining CRM + ERP data with gender fallback logic
- **`dim_products`** — Current product dimension filtered by `prd_end_dt IS NULL`
- **`fact_sales`** — Sales transactions linked to dimensions via surrogate keys

---

## ✅ Data Quality Checks

### Silver Layer Checks
- Primary key uniqueness and null validation
- Whitespace detection in string fields
- Domain value consistency (gender, marital status, product line)
- Date range and chronological order validation
- Mathematical consistency (`sales = quantity × price`)

### Gold Layer Checks
- Surrogate key uniqueness for dimensions
- Referential integrity between fact table and dimensions (orphan record detection)

---

## 📚 Documentation

| Document | Description |
|----------|-------------|
| [Data Catalog](docs/data_catalog.md) | Field-level data dictionary for the Gold Star Schema |
| [Naming Conventions](docs/naming_conventions.md) | Standards for tables, columns, schemas, and procedures |

---

## 🛠️ Tools & Technologies

- **Database:** Microsoft SQL Server Express
- **IDE:** SQL Server Management Studio (SSMS)
- **Architecture:** Medallion (Bronze / Silver / Gold)
- **Data Modeling:** Star Schema (Kimball methodology)
- **Diagrams:** [Draw.io](https://www.drawio.com/)
- **Version Control:** Git + GitHub

---

## 📜 License

This project is licensed under the [MIT License](LICENSE).

---

## 👤 Author

**Sohom Ghosh**

---

## 🙏 Acknowledgments

This project was inspired by the [Data With Baraa](https://www.youtube.com/@datawithbaraa) SQL Data Warehouse course. Original course materials and methodology by **Baraa Khatib Salkini**.
]]>
