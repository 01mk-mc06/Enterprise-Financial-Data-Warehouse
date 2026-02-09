# Enterprise-Financial-Data-Warehouse
Multi-subsidiary financial consolidation system processing messy data from 5+ sources with complex 15+ table joins handling real-world data quality issues.

Sources (all messy):
NetSuite API: missing cost centers, duplicate GL entries, backdated transactions
QuickBooks CSV: UTF-8 encoding errors, inconsistent date formats (MM/DD vs DD/MM), negative amounts as "(1,234.56)"
Excel budget files: merged cells, hidden rows, formulas instead of values, multiple sheets per entity
Bank CSVs: vendor names abbreviated differently each month, FX rates at different times causing mismatches
Payroll system: NULL department assignments, employees moved mid-month, retroactive salary adjustments

Bronze Layer: Raw ingestion preserving all anomalies

Silver Layer - Cleaned & Normalized (10 tables):

dim_chart_of_accounts - 5 level hierarchy, codes changed 2x during year without mapping table
dim_cost_centers - department mergers, name changes, orphaned IDs in transactions
dim_vendors - same vendor 4+ different spellings/IDs across subsidiaries
dim_customers - merged accounts, acquisitions causing duplicate entries
dim_employees - manager hierarchy with circular references (data errors), transfers between entities
dim_projects - spans fiscal years, budget reallocations not logged properly
dim_legal_entities - 3 subsidiaries with different fiscal year-ends
dim_currencies - daily rates from 3 sources (central bank, accounting system, bank feed) - all differ
fact_gl_transactions - 40% missing cost center, 15% have wrong account codes (discovered later)
fact_intercompany - unbalanced entries, late eliminations, currency translation errors

Gold Layer - Complex Joins (8+ table aggregations):

Data Quality Issues Handled:

Fuzzy matching vendors across subsidiaries using Levenshtein distance
Imputing missing cost centers using GL account patterns + historical assignments
Detecting/flagging backdated entries >30 days
Currency triangulation when direct rates unavailable
Reconciling unbalanced intercompany (force balancing with adjustment accounts)
Handling partial period data (month not closed, accruals estimated)
Late-arriving invoices posted to wrong period (audit trail + restatement logic)
Chart of account mapping for mid-year changes (bridge table with effective dates)
