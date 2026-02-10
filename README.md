# Enterprise Financial Data Warehouse - Project Documentation

## Overview
Built production-grade Enterprise Financial Data Warehouse using Azure cloud stack with medallion architecture (Bronze → Silver → Gold). Processed intentionally messy, **synthetically generated** financial data with complex transformations and multi-way joins.

## Tech Stack
- **Ingestion**: Azure Data Factory
- **Storage**: Azure Data Lake Storage Gen2
- **Processing**: Azure Databricks (PySpark)
- **Format**: Delta Lake
- **Catalog**: Unity Catalog (metadata management)
- **Visualization**: Power BI (basic proof of concept)

## Data Sources (Synthetic)
**All data was synthetically generated using Python to simulate real-world data quality issues:**

1. NetSuite Transactions (520 records) - JSON
2. QuickBooks GL (500 records) - CSV
3. Excel Budget/Actuals/Forecast (180 records) - XLSX
4. Bank Statements (300 records) - CSV
5. Payroll (200 records) - CSV

---

## Project Timeline

### Bronze Layer (Raw Data Ingestion)

#### Milestones:
- Generated intentionally messy synthetic data with duplicates, NULLs, format inconsistencies
- Created Azure infrastructure (Resource Group, Storage Account, ADF, Databricks)
- Built 5 ADF copy pipelines with master orchestration
- Created 7 Bronze Delta tables in Unity Catalog
- Data quality report documenting all issues

#### Roadblocks:
- **Key Vault RBAC** → Removed Key Vault, used direct storage authentication
- **Excel multi-sheet handling** → Used Binary format + Databricks pandas parsing
- **Unity Catalog paths** → Required `abfss://` protocol with proper storage configuration

#### Data Quality Issues Preserved:
- 20 duplicate transaction IDs (NetSuite)
- 30% NULL cost centers (NetSuite)
- 3 mixed date formats (QuickBooks)
- Account code typos: 4OOO, 5IOO (QuickBooks)
- Vendor name variations: "Acme Ltd" vs "ACME LIMITED" (Bank)
- 25% NULL departments (Payroll)
- 20% NULL forecasts (Excel)

---

### Silver Layer (Data Cleaning & Transformation)

#### Milestones:

**M1: Dimension Tables**
- Chart of Accounts (5-level hierarchy)
- Cost Centers (with historical tracking)
- Vendor Master (deduplication mapping)
- Currency Exchange Rates (90 days)

**M2-6: Fact Table Cleaning**
- NetSuite: Removed duplicates, imputed NULLs, flagged backdated entries
- QuickBooks: Standardized dates, fixed account typos, cleaned vendor names
- Bank: Deduplicated vendors, converted currencies to USD, categorized transactions
- Payroll: Imputed departments, tracked transfers, identified salary changes
- Budget/Actuals/Forecast: Added fiscal periods, calculated variances, imputed NULLs

**M7: Data Quality Validation**
- Bronze vs Silver comparison reports
- Quality score validation
- Transformation verification

#### Roadblocks:
- **Unity Catalog storage** → Created custom catalog with managed location: `abfss://financial-warehouse@financialwarehouse.dfs.core.windows.net/bronze/catalogs/financial_dwh`
- **DBFS root disabled** → Used Unity Catalog managed tables with `saveAsTable()` instead of external paths
- **Storage authentication** → Unity Catalog handles authentication automatically with external storage

#### Transformations:
- Date standardization: 3 formats → YYYY-MM-DD
- Vendor deduplication: 50+ variations → 20 unique
- Currency conversion: EUR/GBP → USD
- Department imputation: Forward/backward fill
- Account validation: Against dimension table

---

### Gold Layer (Business Analytics)

#### Milestones:

**M1: Consolidated P&L (12-way join)**
- NetSuite + QuickBooks + Budget + Actuals + Forecast + Prior Year
- Chart of Accounts + Cost Centers + Vendors
- YoY growth, budget variance, quality scores

**M2: Cash Flow Statement (10-way join)**
- GL + Bank + Payroll reconciliation
- Operating/Investing/Financing categories
- Monthly cash movement trends

**M3: Working Capital Analysis (9-way join)**
- AR/AP aging (0-30, 31-60, 61-90, 90+ days)
- DSO calculation
- Collection rate analysis

**M4: Departmental Performance (8-way join)**
- Spend vs budget by department
- Headcount and efficiency metrics
- Vendor analysis per department

**M5: Executive Dashboard**
- Single table with 15+ KPIs
- Revenue, expenses, profit, cash flow
- Operational metrics (headcount, DSO, collection rate)
- Monthly trends and YTD summaries

**M6: Final Validation**
- Complete layer summary
- Record counts and quality scores
- Business metrics validation

#### Roadblocks:
- **Missing column reference** → Removed `cc.division` - column didn't exist in dimension table
- **SQL syntax error** → Fixed UNION ALL query formatting

---

## Final Architecture

### Bronze Layer (7 tables)
Raw, unprocessed data with all quality issues preserved:
- netsuite_transactions
- quickbooks_gl
- budget, actuals, forecast
- bank_statements
- payroll

### Silver Layer (11 tables)
Cleaned, validated, standardized data:

**Facts:** 7 cleaned transaction tables  
**Dimensions:** 4 reference tables (accounts, cost centers, vendors, fx rates)

### Gold Layer (5 tables)
Pre-aggregated, business-ready analytics tables:
- consolidated_pl
- cash_flow_statement
- working_capital
- departmental_performance
- executive_dashboard

---

## Unity Catalog Implementation

### What Unity Catalog Provided:
- **3-level namespace**: `catalog.schema.table` organization structure
- **Metadata management**: Table definitions, schemas, and locations tracked centrally
- **External storage integration**: Tables stored in ADLS Gen2, Unity Catalog maintains pointers and metadata

### Storage Architecture:
```
ADLS Gen2 (Physical Storage):
abfss://financial-warehouse@financialwarehouse.dfs.core.windows.net/
└── bronze/catalogs/financial_dwh/
    ├── bronze/ (Delta tables)
    ├── silver/ (Delta tables)
    └── gold/ (Delta tables)

Unity Catalog (Metadata Layer):
financial_dwh (catalog)
├── bronze (schema)
├── silver (schema)
└── gold (schema)
```

### What Was NOT Implemented:
**Governance features were not utilized in this project:**
- No access control policies
- No data lineage tracking
- No audit logging
- No fine-grained permissions
- No data classification/tagging
- No cross-workspace sharing

**Caveat:** Unity Catalog was used solely for metadata management and table organization. For a single-workspace learning project, the default Hive metastore would have been simpler. Unity Catalog's governance features are valuable for multi-workspace enterprise deployments but were overkill for this scope.

---

## Power BI Integration

### Implementation Status: Basic Proof of Concept

**What Was Implemented:**
- Direct connection to Databricks
- Import of Gold layer tables
- Basic visualizations (cards, line charts, tables)
- Validation that data warehouse outputs are analytics-ready

### Caveats - What Was NOT Implemented:

**1. Data Modeling:**
- No star schema relationships defined in Power BI
- No dimension table imports from Silver layer
- No proper fact-to-dimension connections
- Each Gold table treated as isolated dataset

**2. DAX Measures:**
- No time intelligence (YTD, QTD, MTD)
- No dynamic rankings or comparative measures
- No calculated columns or measure tables
- All metrics pre-calculated in Gold layer

**3. Advanced Features:**
- No drill-through between reports
- No cascading filters across tables
- No what-if parameters
- No row-level security

### Design Decision:
The project prioritized **data engineering over visualization**. All complex transformations, joins, and business logic were implemented in the data warehouse (PySpark/SQL), making Power BI a simple presentation layer. This follows a "Gold layer does heavy lifting" architecture where:

-  Complex metrics pre-calculated in data warehouse [Done]
-  Single source of truth for business logic [Done]
-  Reusable across multiple BI tools [Done]
-  Less flexibility for ad-hoc analysis in Power BI [X]
-  Requires data engineering changes for new metrics [X]

**For a complete BI solution, proper Power BI modeling and DAX measures should be added.**

---

## Key Achievements
- **22 total tables** across 3 layers
- **Complex joins**: 12-way, 10-way, 9-way, 8-way
- **Data quality**: 100% NULL imputation, duplicate removal, format standardization
- **Multi-source**: 5 disparate sources unified
- **Synthetic data**: Realistic data quality scenarios generated programmatically

---

## Known Limitations

### 1. Synthetic Data
All data was generated using Python scripts to simulate real-world scenarios. While realistic in structure and quality issues, it lacks:
- Actual business context
- Real vendor/customer relationships
- Historical trends and seasonality
- Regulatory compliance requirements

### 2. Unity Catalog Utilization
Used only for basic metadata management. Governance features (access control, lineage, audit logs) not implemented. For single-workspace projects, default Hive metastore would be simpler.

### 3. Power BI Implementation
Basic connection only. Missing proper data modeling (star schema, relationships) and DAX measures (time intelligence, dynamic calculations). Suitable for proof of concept but not production dashboard.

### 4. Scalability Not Tested
Pipeline designed for ~2,000 records total. Performance at scale (millions of rows) not validated. Optimizations like partitioning, caching, and incremental loads not implemented.

### 5. Orchestration
Manual execution of notebooks. No production scheduling, error handling, alerting, or SLA monitoring implemented.

---

## Portfolio Highlights
- Medallion architecture implementation
- Complex data quality scenario handling
- Multi-way joins (up to 12 tables)
- Azure cloud stack expertise
- Delta Lake format implementation
- Unity Catalog metadata management
- End-to-end data pipeline (ingestion → analytics)

---

## Future Enhancements
1. **Power BI**: Implement proper star schema modeling and DAX measures
2. **Orchestration**: Add production scheduling with monitoring/alerting
3. **Incremental Loads**: Implement CDC for efficient daily refreshes
4. **Data Quality**: Add automated testing framework (Great Expectations)
5. **Governance**: Utilize Unity Catalog access controls and lineage tracking
6. **Real Data**: Integrate with actual financial systems (with proper anonymization)
7. **Machine Learning**: Add forecasting models for budget predictions
