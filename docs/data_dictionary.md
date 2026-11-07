# Data Dictionary

This document explains the columns, rules, and table grain used in my Online Retail Sales Analysis project. I wrote it from the data I loaded and checked in Databricks.

## Dataset source

I used the UCI Machine Learning Repository’s [Online Retail dataset](https://archive.ics.uci.edu/dataset/352/online%2Bretail). It contains transaction records for a UK-based online retailer from December 1, 2010 to December 9, 2011. The dataset documentation describes `UnitPrice` in sterling (GBP).

The original file has 541,909 rows. I converted it to CSV for Databricks ingestion.

## Source columns

The Bronze table stores the original source fields as strings so I can inspect the raw values before converting them.

| Column | Bronze type | Silver type | What it represents |
|---|---|---|---|
| `InvoiceNo` | STRING | STRING | Invoice identifier. A value beginning with `C` indicates a cancellation. |
| `StockCode` | STRING | STRING | Code identifying a product. It can contain letters, so I keep it as text. |
| `Description` | STRING | STRING | Product description. Missing values remain `NULL`. |
| `Quantity` | STRING | INT | Number of items on the transaction line. Negative values are preserved. |
| `InvoiceDate` | STRING | TIMESTAMP | Date and time when the transaction was generated. |
| `UnitPrice` | STRING | DECIMAL(18,4) | Price per item in sterling. Zero and negative values are preserved. |
| `CustomerID` | STRING | STRING | Customer identifier. Missing values remain `NULL`. |
| `Country` | STRING | STRING | Country associated with the customer. |
| `_rescued_data` | STRING | Not included | Technical field added during ingestion. It had no non-`NULL` values in my Bronze table. |

I parsed `InvoiceDate` using the source format `M/d/yyyy H:mm`. The source does not specify a timezone, so I do not treat the timestamp as proof of a particular timezone.

## Silver columns

My Silver table is `workspace.retail_project.silver_online_retail`.

It has one row for every Bronze row. In addition to the converted source fields, it contains two flags:

| Column | Type | Definition |
|---|---|---|
| `is_cancellation` | BOOLEAN | `TRUE` when the invoice number starts with `C`, after trimming whitespace and checking without regard to letter case. |
| `is_negative_quantity` | BOOLEAN | `TRUE` when `Quantity` is less than zero. |

These flags describe different things. A row can be marked as a cancellation, have a negative quantity, or both. I did not remove rows, replace missing values, or change negative quantities in Silver.

## Gold tables

### `gold_sales_summary`

Despite the table name, this is a **line-level sales table**, not an aggregated summary. It contains one row per included sales line.

**Inclusion rule:** `Quantity > 0` and `is_cancellation = FALSE`.

It includes these columns from Silver:

| Column | Meaning |
|---|---|
| `InvoiceNo` | Invoice associated with the sales line. |
| `StockCode` | Product code. |
| `Description` | Product description; may be `NULL`. |
| `Quantity` | Positive quantity on the included sales line. |
| `InvoiceDate` | Timestamp for the transaction. |
| `UnitPrice` | Unit price in sterling. |
| `CustomerID` | Customer identifier; may be `NULL`. |
| `Country` | Customer country. |

I did not add a condition requiring `UnitPrice` to be positive. Therefore, a row can qualify for this sales table because its quantity is positive even if its unit price is zero or negative.

### Aggregated Gold tables

| Table | One row represents | Main measures |
|---|---|---|
| `gold_sales_daily` | One calendar date | Sales line count, distinct invoice count, units sold, revenue |
| `gold_sales_monthly` | One calendar month | Sales line count, distinct invoice count, units sold, revenue |
| `gold_sales_by_country` | One country | Sales line count, distinct invoice count, units sold, revenue |
| `gold_sales_by_stockcode` | One StockCode | Sales line count, distinct invoice count, units sold, revenue |

All four tables are built from `gold_sales_summary`, so they use the same sales inclusion rule.

## Metric definitions

| Metric | Definition |
|---|---|
| **Sales line count** | Number of included rows in the line-level sales table. |
| **Sales invoices** | Number of distinct `InvoiceNo` values among included sales lines. |
| **Units sold** | Sum of `Quantity` for included sales lines. |
| **Revenue** | Sum of `Quantity × UnitPrice` for included sales lines. The source currency is GBP. |
| **Average Order Value** | Revenue divided by the number of distinct sales invoices. |

“Sales lines,” “invoices,” and “units sold” are different measures: a single invoice can contain multiple sales lines, and each line can contain multiple units.

## Data quality notes

My checks found:

- Bronze and Silver each contain 541,909 rows.
- Silver has no `NULL` values in `Quantity`, `UnitPrice`, or `InvoiceDate`.
- Silver contains 135,080 `NULL` values in `CustomerID`.
- Silver contains 1,454 `NULL` values in `Description`.
- Quantity has 10,624 negative rows. Of these, 9,288 have a `C`-prefixed invoice number.
- The source period ends on December 9, 2011, so December 2011 is an incomplete month.

The UCI dataset page says the dataset has no missing values, but my loaded copy contains missing customer IDs and descriptions. I recorded what I observed in my data rather than filling in those values or assuming why the source differs.

I also reconciled the daily, monthly, country, and StockCode Gold tables against the line-level sales table. Their units and revenue totals matched.

## Dashboard

My Databricks dashboard has eight widgets:

**KPI cards**

- Total Revenue
- Units Sold
- Average Order Value
- Sales Invoices

**Charts**

- Top five countries by revenue
- Revenue per month
- Monthly units sold
- Top ten StockCodes by revenue

The dashboard uses compact number formatting. Revenue and Average Order Value should be displayed with a GBP currency format so the currency is clear.

## Attribution

Chen, D. (2015). *Online Retail* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C5BW33

The dataset is licensed under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).