# Retail Data Warehouse — Star Schema Design in Power BI

A dimensional data modeling project that transforms 23 raw, multi-source retail tables — customer, product, order, invoice, shipment, and marketing campaign data — into a clean, analysis-ready star schema using **Power Query** and **DAX** in Power BI.

Unlike a typical dashboard project, the focus here is entirely on the **data modeling layer**: taking disconnected, inconsistent source tables and engineering them into a proper Kimball-style star schema, complete with a junk dimension, a role-playing dimension, a factless fact table, an accumulating snapshot fact table, and row-level security.

## Project Overview

This project takes that raw extract and builds a single, unified semantic model in Power BI capable of answering business questions across sales, inventory, marketing, and order fulfillment — all from one shared date dimension.

## Objectives

- Consolidate scattered, duplicated, and inconsistently structured source tables into a single source of truth
- Design a star schema that avoids fact-to-fact relationships and filters correctly from dimensions to facts
- Model multiple fact table patterns (transactional, snapshot, factless, accumulating snapshot) within one schema
- Track the full order lifecycle (order → ship → deliver → invoice → pay) in a single row per order
- Secure the model with region-based Row-Level Security (RLS)
- Apply consistent naming standards and clean, load-optimized tables (no unused staging tables left behind)

## The Raw Data: A Deliberately Messy Multi-Source Extract

The source workbook contains **23 raw tables**, engineered to reflect common real-world data problems:

| Issue | Example |
|---|---|
| **Schema drift across years** | `ORDERS_2025` has extra columns (`LegacyRef`, `OrderNotes`, `GiftMessage`, `SourceFile`) that `ORDERS_2026` doesn't — the two must be reconciled before appending |
| **Wide/pivoted format** | `inventory` stores 12 months of stock as separate columns (`2025-01` … `2025-12`) instead of rows — needs unpivoting into a proper snapshot fact |
| **Delimited list fields** | `campaign_skus.PromotedSKUs` packs multiple product codes into one comma-separated cell per campaign — needs splitting into one row per SKU |
| **Repeated header-level data** | `CAMPAIGN_LOG` repeats `Budget`, `StartDate`, and `EndDate` on every daily row — campaign attributes need separating from daily performance metrics |
| **Duplicate/accidental tables** | `shipments` and `Sheet1` are identical in structure — a classic leftover duplicate import |
| **Source-system artifacts** | `CUST_MASTER` and `products` carry `hash_key` / `source_id` columns simulating an upstream SAP-style extract |
| **Test/dummy records** | `CUST_MASTER` includes a `9999 – TEST ACCOUNT` row that has to be filtered out |
| **Normalized-away geography** | City and region live in separate `Address`, `cities`, and `regions` tables rather than one clean geo dimension |
| **Unused noise tables** | `user_details`, `exchange_rates`, and a leftover single-column `dim_order` reference weren't needed in the final model and were reviewed and excluded |

## Data Modeling Process

1. **Customer dimension** — merged `CUST_MASTER`, `customer_contacts`, and `Address` into a single reference table, then cleaned headers and removed unused columns to build `dim_customer`.
2. **Product dimension** — merged `products` with `subcategories` (splitting the combined category/subcategory string) to build `dim_product`.
3. **Sales fact** — appended `ORDERS_2025` and `ORDERS_2026`, resolved the schema drift between them, then merged in `order_line_items` to build the transactional-grain `fact_sales`. A **junk dimension** (`dim_order_flags`) was split out to hold low-cardinality order attributes (status, priority, channel) separately from the fact.
4. **Geography as a role-playing dimension** — built one `dim_geo` table from `cities` + `regions`, then connected it **twice** to `fact_sales` (Ship-To City and Bill-To City), with only one relationship set active at a time to avoid ambiguous filter paths.
5. **Inventory fact** — unpivoted the wide monthly `inventory` table and merged it with `dim_product` to build a periodic snapshot fact, `fact_inventory`.
6. **Campaign facts** — split `CAMPAIGN_LOG` into a clean `dim_campaign` (one row per campaign) and a daily performance fact, `fact_campaign_spend`. Separately, `campaign_skus` was unpacked (one row per promoted SKU) and merged with `dim_campaign` and `dim_product` to build `fact_promotion_coverage` — a **factless fact table** used purely to answer "which products were covered by which campaign."
7. **Accumulating snapshot fact** — built `fact_order_process` by referencing `orders` and merging in every downstream milestone date (order, ship, delivery, invoice, payment) from `shipments`, `invoices`, and `payments`, producing one row per order with every stage's date as its own column.
8. **Sales targets fact** — loaded `sales_targets` directly as `fact_sales_targets`.
9. **Standards pass** — enforced `snake_case` naming, `fact_`/`dim_` table prefixes, `_key`/`_id` column suffixes, and standardized date formats across every table before finalizing relationships.
10. **Shared date dimension** — built `dim_date` (via `CALENDARAUTO()` plus Year/Month columns) and connected it to every fact table containing dates, enabling consistent time-intelligence and cross-fact reporting from one dimension.
11. **Row-Level Security** — related the `security` table to `dim_customer` by region and applied a DAX-based RLS role so users only see data for their assigned region.

## Final Data Model (Star Schema)

**6 Dimensions:** `dim_customer` · `dim_product` · `dim_geo` (role-playing) · `dim_order_flags` (junk) · `dim_campaign` · `dim_date` (shared/conformed)

**6 Fact Tables:** `fact_sales` (transactional) · `fact_inventory` (periodic snapshot) · `fact_campaign_spend` (transactional) · `fact_promotion_coverage` (factless) · `fact_order_process` (accumulating snapshot) · `fact_sales_targets`

**Plus:** a `security` table driving region-based RLS on `dim_customer`

*(23 raw source tables consolidated into this 13-table model.)*

## Key Data Modeling Concepts Applied

- ⭐ **Star schema design** — dimensions connect only to facts, never fact-to-fact; a shared `dim_date` links every fact for cross-subject reporting
- 🧩 **Junk dimension** — low-cardinality order flags grouped into one small dimension instead of bloating the fact table
- 🔁 **Role-playing dimension** — a single `dim_geo` reused for both Ship-To and Bill-To City via one active and one inactive relationship
- 🚫 **Factless fact table** — `fact_promotion_coverage` records a relationship (campaign ↔ product) with no numeric measure
- 📈 **Accumulating snapshot fact** — `fact_order_process` holds every milestone date of the order lifecycle in a single, updatable row
- 🔒 **Row-Level Security (RLS)** — DAX-based region filtering from `security` through `dim_customer`
- 📐 **Naming & load standards** — consistent `snake_case`, `fact_`/`dim_` prefixes, `_key`/`_id` suffixes, and disabled load on every staging/reference table once merged

## Report View

The report canvas includes a summary table sourced from `dim_date`'s Year/Quarter/Month hierarchy alongside `SUM(target_revenue)`, `SUM(units)`, and `SUM(line_total)` — a lightweight proof that every fact table filters correctly through the shared date dimension. The real deliverable of this project is the model itself, so most of the value is in the **Model view** (star schema layout) rather than the report canvas.

## Tools & Technologies

- Power BI Desktop
- Power Query (M) — data transformation and cleaning
- DAX — RLS roles and summary measures
- Dimensional Modeling (Kimball methodology)

## Key Learnings

- **Explore before you build** — understand the business context, the source processes, and the data itself before touching a single table
- **Know the grain** — don't connect tables blindly; every merge should have a clear, deliberate grain in mind
- **Set standards up front** — naming and formatting conventions are far easier to enforce from the start than to retrofit
- **Remove what you don't need** — unused staging tables and reference data (e.g. `user_details`, `exchange_rates`) were reviewed and left out of the final model
- **Build the star, not a snowflake or a web** — never connect fact tables directly; route every relationship through a shared dimension
- **Filter direction matters** — relationships should always filter from dimensions to facts, keeping the model predictable and performant

## Project Structure

```text
Retail-Data-Warehouse-PowerBI/
│
├── Data/
│   └── dataset.xlsx
│
├── PowerBI/
│   └── Retail_Data_Warehouse.pbix
│
└── README.md
```

## About the Data

The dataset is a synthetically generated retail extract created for the purpose of practicing real-world data modeling scenarios (multi-source structure, schema drift, wide formats, delimited fields, duplicate/test records). All company, customer, and product names are fictional.
