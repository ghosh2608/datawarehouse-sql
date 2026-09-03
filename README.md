# Data Warehouse & Analytics Project

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![SQL Server](https://img.shields.io/badge/SQL%20Server-Express-CC2927?logo=microsoftsqlserver&logoColor=white)](https://www.microsoft.com/en-us/sql-server/sql-server-downloads)
[![Architecture](https://img.shields.io/badge/Architecture-Medallion-gold)](docs/data_architecture.png)

A production-style **Data Warehouse** built on Microsoft SQL Server, implementing the **Medallion Architecture** (Bronze → Silver → Gold) to consolidate ERP and CRM data into a Star Schema optimized for analytics and BI reporting.

---

## Architecture

This project follows the **Medallion Architecture** pattern with three distinct layers:

| Layer | Purpose | Implementation |
|:------|:--------|:---------------|
| **Bronze** | Raw data ingestion — stores data as-is from source systems | `BULK INSERT` from CSV files, full truncate-and-reload |
| **Silver** | Cleansed, standardized, deduplicated, and enriched data | Stored procedure with business rules, type conversions, and null handling |
| **Gold** | Business-ready dimensional model for analytics | SQL Views implementing a Star Schema with surrogate keys |

> **Source Systems:** CRM (customer info, products, sales) and ERP (locations, demographics, categories) provided as CSV files.

---

## Data Model (Gold Layer)

The Gold layer implements a **Star Schema** with two dimension views and one fact view:

**`gold.dim_customers`** — Customer dimension enriched from CRM + ERP with gender fallback logic

**`gold.dim_products`** — Product dimension filtered for current products only

**`gold.fact_sales`** — Sales transactions linked to dimensions via surrogate keys

| View | Type | Key Fields |
|:-----|:-----|:-----------|
| `dim_customers` | Dimension | `customer_key` (PK), `customer_id`, `first_name`, `last_name`, `country`, `gender`, `marital_status`, `birthdate` |
| `dim_products` | Dimension | `product_key` (PK), `product_number`, `product_name`, `category`, `subcategory`, `cost`, `product_line` |
| `fact_sales` | Fact | `order_number`, `customer_key` (FK), `product_key` (FK), `order_date`, `sales_amount`, `quantity`, `price` |

For full field descriptions and data types, see the [Data Catalog](docs/data_catalog.md).

---

## Project Structure

```
datawarehouse_project/
│
├── datasets/
│   ├── source_crm/              # CRM exports (customers, products, sales)
│   └── source_erp/              # ERP exports (locations, demographics, categories)
│
├── scripts/
│   ├── init_database.sql        # Database & schema initialization
│   ├── bronze/
│   │   ├── ddl_bronze.sql       # Bronze table definitions (6 tables)
│   │   └── proc_load_bronze.sql # Bulk ingestion stored procedure
│   ├── silver/
│   │   ├── ddl_silver.sql       # Silver table definitions
│   │   └── proc_load_silver.sql # Cleansing & transformation procedure
│   └── gold/
│       └── ddl_gold.sql         # Star Schema views (2 dims + 1 fact)
│
├── tests/
│   ├── quality_checks_silver.sql
│   └── quality_checks_gold.sql
│
├── docs/
│   ├── data_catalog.md          # Gold layer data dictionary
│   └── naming_conventions.md    # Naming standards
│
├── LICENSE
└── README.md
```

---

## Getting Started

### Prerequisites

- [SQL Server Express](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) — free database engine
- [SQL Server Management Studio (SSMS)](https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms) — GUI for executing scripts

### Setup

Run the scripts **in order** within SSMS:

| Step | Script | What It Does |
|:-----|:-------|:-------------|
| 1 | `scripts/init_database.sql` | Creates `DataWarehouse` database and `bronze`, `silver`, `gold` schemas |
| 2 | `scripts/bronze/ddl_bronze.sql` | Creates 6 Bronze landing tables |
| 3 | `scripts/bronze/proc_load_bronze.sql` | Creates and runs `bronze.load_bronze` — bulk loads CSV data |
| 4 | `scripts/silver/ddl_silver.sql` | Creates 6 Silver tables with typed columns and audit fields |
| 5 | `scripts/silver/proc_load_silver.sql` | Creates and runs `silver.load_silver` — cleanses and transforms data |
| 6 | `scripts/gold/ddl_gold.sql` | Creates Star Schema views (`dim_customers`, `dim_products`, `fact_sales`) |
| 7 | `tests/quality_checks_silver.sql` | *(Optional)* Validates Silver layer data quality |
| 8 | `tests/quality_checks_gold.sql` | *(Optional)* Validates Gold layer integrity |

> **Important:** Update the file paths in `proc_load_bronze.sql` to match your local CSV file locations before running Step 3.

---

## ETL Pipeline

### Bronze Layer — Raw Ingestion

- **Strategy:** Full truncate-and-reload
- **Method:** `BULK INSERT` from CSV files (skips header row)
- **Tables:** 6 tables mirroring CRM and ERP source structures
- **Error Handling:** `TRY...CATCH` with per-table load timing

### Silver Layer — Cleansing & Transformation

Key transformations performed by `silver.load_silver`:

- **Deduplication** — `ROW_NUMBER()` partitioned by customer ID, keeping the latest record
- **String Sanitization** — `TRIM()` on name fields to remove whitespace
- **Value Normalization** — Code-to-label mapping (e.g., `'S'` → `'Single'`, `'M'` → `'Mountain'`)
- **Date Validation** — Rejects invalid/future dates, converts integer dates to `DATE` type
- **Business Rule Enforcement** — Recalculates `sales_amount` when inconsistent with `quantity × price`
- **Key Parsing** — Extracts category ID and product key from composite keys
- **Country Standardization** — Maps codes (`'DE'`, `'US'`, `'USA'`) to full country names
- **Gender Harmonization** — Normalizes across CRM and ERP sources with fallback logic

### Gold Layer — Star Schema

- **`dim_customers`** — Enriched customer dimension joining CRM + ERP data with gender fallback logic
- **`dim_products`** — Current product dimension filtered by `prd_end_dt IS NULL`
- **`fact_sales`** — Sales transactions linked to dimensions via surrogate keys

---

## Data Quality Checks

### Silver Layer

- Primary key uniqueness and null validation
- Whitespace detection in string fields
- Domain value consistency (gender, marital status, product line)
- Date range and chronological order validation
- Mathematical consistency (`sales = quantity × price`)

### Gold Layer

- Surrogate key uniqueness for dimensions
- Referential integrity between fact table and dimensions (orphan record detection)

---

## Documentation

- [Data Catalog](docs/data_catalog.md) — Field-level data dictionary for the Gold Star Schema
- [Naming Conventions](docs/naming_conventions.md) — Standards for tables, columns, schemas, and procedures

---

## Tools & Technologies

- **Database:** Microsoft SQL Server Express
- **IDE:** SQL Server Management Studio (SSMS)
- **Architecture:** Medallion (Bronze / Silver / Gold)
- **Data Modeling:** Star Schema (Kimball methodology)
- **Diagrams:** [Draw.io](https://www.drawio.com/)
- **Version Control:** Git + GitHub

---

## License

This project is licensed under the [MIT License](LICENSE).

---

## Author

**Sohom Ghosh**

---

## Acknowledgments

This project was inspired by the [Data With Baraa](https://www.youtube.com/@datawithbaraa) SQL Data Warehouse course. Original course materials and methodology by **Baraa Khatib Salkini**.
