# DSA3050-PowerBI-Bophine_Arnold_Odiyo-668821
# DSA 3050A — Business Intelligence & Data Visualization
## End Semester Practical Examination — Power BI Solution

**Student Name:** Bophine Arnold Odiyo
**Registration Number:** 668821
**Dataset:** Global Superstore (`data/dataset.csv`)
**Software:** Microsoft Power BI Desktop
**Submission Date:** 15/08/2026



## Table of Contents
1. [Dataset Selection & Understanding](#1-dataset-selection--understanding)
2. [Power Query — Data Cleaning & Transformation](#2-power-query--data-cleaning--transformation)
3. [Data Modelling](#3-data-modelling)
4. [DAX Measures & Business Calculations](#4-dax-measures--business-calculations)
5. [Dashboard Design](#5-dashboard-design)
6. [Key Analytical Results](#6-key-analytical-results)
7. [Repository Structure](#7-repository-structure)



## 1. Dataset Selection & Understanding

**Source:** Global Superstore, a widely used open BI teaching dataset. [State exactly where you downloaded your copy — Kaggle link or equivalent.]

**What it represents:** Each row is a single line item within a customer order (one order can contain multiple line items if several products were purchased together), covering orders placed globally between 2011 and 2014. The dataset records sales revenue, profit, discount applied, quantity, shipping cost, and shipping performance for every line item, alongside customer, product, and geographic attributes.

**Why selected:** The dataset contains 51,290 records — well above the assignment's 20,000-row minimum — spans 147 countries and four years, and contains genuine, non-trivial profitability problems (a meaningful share of orders lose money), making it suitable for real diagnostic BI analysis rather than purely descriptive reporting.

**Main variables:**
- **Numerical:** `Sales`, `Profit`, `Discount`, `Quantity`, `Shipping.Cost`
- **Categorical:** `Category`, `Sub.Category`, `Segment`, `Market`, `Region`, `Country`, `State`, `City`, `Ship.Mode`, `Order.Priority`
- **Date/time:** `Order.Date`, `Ship.Date`, `Year`, `weeknum`
- **Identifiers:** `Order.ID`, `Customer.ID`, `Product.ID`, `Row.ID`

**Business/analytical problem:** The company wants to understand which markets, products, and customer segments are profitable versus loss-making, and specifically why shipping performance and discounting behaviour appear to be eroding margins in certain regions and product lines.

**Analytical questions this solution answers:**
1. Which product categories and sub-categories drive the most profit, and which lose money?
2. How does discount level affect profit margin and loss-order rate across the business?
3. Which markets and regions have the strongest and weakest profitability?
4. Is there a meaningful year-over-year sales and profit trend, and is profit growth keeping pace with sales growth?
5. Does shipping mode or order priority correlate with shipping duration or profitability?
6. Which products/sub-categories should be prioritized for a pricing or discount policy review?


## 2. Power Query — Data Cleaning & Transformation

The raw file was imported via **Get Data → Text/CSV → Transform Data**, landing directly in the Power Query Editor (not loaded as-is). The following transformations were applied, each addressing a specific problem identified in the raw data.
<img width="1910" height="1070" alt="Screenshot 2026-08-15 101233" src="https://github.com/user-attachments/assets/79395800-e4e2-4622-a47e-017d6455a2cf" />
<img width="1919" height="1076" alt="01_raw_data" src="https://github.com/user-attachments/assets/7f198d6e-edef-4a08-827e-c0278fea00ca" />

### Transformation 1 — Removing junk column
**Problem:** The column `记录数` contained the constant value `1` for every row and carried no analytical meaning.

**Transformation:** Right-click the column → Remove.

**Reason:** A constant column adds no information and pollutes the model.

**Result:** Table reduced from 27 to 26 useful columns.

### Transformation 2 — Resolving redundant geography column
**Problem:** `Market2` duplicated `Market` at a coarser grain (e.g. `Market2 = "North America"` when `Market = "US"` or `"Canada"`).

**Transformation:** Retained `Market2`, renamed to `MarketGroup`, and kept it as a second, coarser level of a Market → MarketGroup hierarchy in `DimLocation`.

**Reason:** Rather than discard information, the two fields form a legitimate two-level geography hierarchy once clearly distinguished and renamed.

**Result:** A clean Market/MarketGroup hierarchy instead of two ambiguous, seemingly duplicate fields.

### Transformation 3 — Fixing data types across all columns
**Problem:** Several columns imported with incorrect or ambiguous types — dates as text, IDs at risk of being auto-detected as numeric, currency fields needing explicit decimal typing.

**Transformation:** Set `Order.Date`/`Ship.Date` to **Date**; `Sales`, `Profit`, `Discount`, `Shipping.Cost` to **Decimal Number**; `Quantity`, `Row.ID`, `Year`, `weeknum` to **Whole Number**; and manually verified all four ID fields (`Order.ID`, `Customer.ID`, `Product.ID`, `Row.ID` excepted, as it's a genuine index) are locked to **Text**.

**Reason:** Power BI cannot perform date intelligence or correct aggregation on text-typed fields, and ID fields must never be summed/averaged or have leading characters silently altered by numeric typing.

**Result:** All fields carry correct, model-ready data types; ID fields are protected against future refresh-time misclassification.

### Transformation 4 — Renaming fields to a consistent naming convention
**Problem:** Column names used dot-notation (`Order.Date`, `Customer.ID`, `Sub.Category`), inconsistent with standard Power BI/DAX naming and harder to reference in formulas.

**Transformation:** Renamed to PascalCase: `OrderDate`, `ShipDate`, `CustomerID`, `ProductID`, `OrderID`, `SubCategory`, `OrderPriority`, `ShipMode`, `ShippingCost`, `RowID`.

**Reason:** Clean, consistent naming is easier to reference in DAX and reads more professionally in visuals.

**Result:** All fields follow one naming convention throughout the model.

### Transformation 5 — Creating Shipping Duration column
**Problem:** No direct measure of delivery speed existed, despite both order and ship dates being present.

**Transformation:** Custom column: `ShippingDurationDays = Duration.Days([ShipDate] - [OrderDate])`.

**Reason:** Enables analysis of whether specific regions, ship modes, or priority levels experience slower fulfilment.

**Result:** New whole-number column, typically ranging 0–7 days.

### Transformation 6 — Creating Profitability Flag column
**Problem:** No categorical field existed to quickly filter or count loss-making versus profitable line items.

**Transformation:** Conditional column: `ProfitFlag = if [Profit] < 0 then "Loss" else if [Profit] = 0 then "Break-even" else "Profit"`.

**Reason:** Enables fast categorical slicing and counting for the diagnostic dashboard page.

**Result:** New text column with three categories, used throughout Section 3 of the analysis.

### Transformation 7 — Creating Discount Band column
**Problem:** `Discount` is a continuous decimal (0–0.85), difficult to use directly in bar charts or slicers.

**Transformation:** Conditional column grouping discount into `No Discount`, `Low (0-20%)`, `Medium (20-50%)`, `High (50%+)`.

**Reason:** Groups a continuous variable into business-meaningful bands for clearer visual analysis of discount impact on margin.

**Result:** New categorical column, central to the discount-vs-profitability diagnostic finding.

### Transformation 8 — Spliting OrderID into component parts
**Problem:** `OrderID` (e.g. `SU-2012-6840`) packs a market/region prefix, order year, and sequence number into one text field, making these embedded attributes unfilterable.

**Transformation:** Split by delimiter (`-`) into `OrderMarketCode`, `OrderYearFromID`, `OrderSequence`.

**Reason:** Exposes the embedded region code as a filterable attribute and allows cross-validation against `OrderDate`'s year.

**Result:** Three new columns; confirmed 100% pattern consistency across all 51,290 rows.

### Transformation 9 — Verifying and lock ID column types
**Problem:** Power BI's type auto-detection can silently misclassify ID columns as numeric, which breaks joins and can strip meaningful leading characters.

**Transformation:** Manually verified and explicitly locked all ID columns (`OrderID`, `CustomerID`, `ProductID`) to Text type.

**Reason:** IDs are labels, not quantities, and should never be summed/averaged; locking the type prevents future data refreshes from silently reclassifying them.

**Result:** Stable relationship keys across refreshes.

### Transformation 10 — Removing duplicate rows
**Problem:** Data integrity needed to be verified before modelling.

**Transformation:** Select all columns → Remove Rows → Remove Duplicates.

**Reason:** Standard data-quality check prior to building relationships.

**Result:** 0 duplicate rows found, confirming the source data's integrity.

<img width="1913" height="1078" alt="02_power_query" src="https://github.com/user-attachments/assets/d1d076e2-de78-4df7-a808-b3055bf40720" />



## 3. Data Modelling

### Model Explanation

The data was transformed from a single flat table into a **star schema** consisting of one fact table and four dimension tables, connected by one-to-many relationships.

**FactSales** contains transactional information and forms the centre of the model. Each row represents one order line item, carrying the transaction's foreign keys (`OrderID`, `OrderDate`, `CustomerID`, `ProductID`, `LocationKey`) alongside all numeric measures (`Sales`, `Profit`, `Discount`, `Quantity`, `ShippingCost`, `ShippingDurationDays`) and transaction-level flags (`ProfitFlag`, `DiscountBand`).

**`DimDate`**, **`DimCustomer`**, **`DimProduct`**, and **`DimLocation`** provide descriptive attributes used to filter and group the fact table. One-to-many relationships were established between each dimension and the fact table, with `DimDate` additionally marked as the model's official Date Table to support time-intelligence calculations.

### Why FactSales was selected as the fact table

`FactSales` holds the transactional grain of the dataset — one row per order line item — and contains every numeric value that needs to be aggregated (Sales, Profit, Discount, Quantity, Shipping Cost). It is the largest table (51,290 rows) and the natural centre of the model, since every analytical question in this project ultimately reduces to summing, averaging, or comparing values from this table, filtered by one or more dimensions.

### Why each dimension was created

- **DimDate** was created because the fact table's raw date fields (`OrderDate`, `ShipDate`) alone cannot support time-intelligence functions like `SAMEPERIODLASTYEAR` or produce chronologically-sorted axes without a dedicated, continuous calendar table. `DimDate` was built as a complete continuous calendar from 2011-01-01 to 2014-12-31, rather than only the distinct dates present in the order data, to avoid gaps in trend visuals and ensure accurate YoY calculations.
  
- **DimCustomer** was created to hold customer-level attributes (`CustomerName`, `Segment`) once per unique customer, rather than repeating this text on every one of that customer's order lines. This supports customer-level and segment-level analysis without redundant storage.
  
- **DimProduct** was created to hold product-level attributes (`ProductName`, `Category`, `SubCategory`) once per unique product, enabling the product-profitability analysis (Section 2 of the accompanying BI report) without repeating category/sub-category text across every transaction involving that product.
  
- **DimLocation** was created to hold geographic attributes (`Country`, `State`, `City`, `Region`, `Market`, `MarketGroup`) once per unique location combination, supporting the regional and market-level diagnostic analysis.

### Relationships used

| From (Dimension) | To (Fact) | Purpose |
|---|---|---|
| `DimDate[Date]` | `FactSales[OrderDate]` | Enables filtering/aggregating sales and profit by year, quarter, month |
| `DimCustomer[CustomerID]` | `FactSales[CustomerID]` | Enables filtering/aggregating by customer and segment |
| `DimProduct[ProductID]` | `FactSales[ProductID]` | Enables filtering/aggregating by category, sub-category, product |
| `DimLocation[LocationKey]` | `FactSales[LocationKey]` | Enables filtering/aggregating by country, region, market |

`DimLocation` has no single natural key in the source data, so a composite key was engineered during Power Query: `LocationKey = [Country] & "|" & [State] & "|" & [City]`, applied identically in both `DimLocation` and `FactSales` to enable the join.

### Cardinality decisions

All four relationships are **one-to-many** (1:*), dimension to fact. This reflects the underlying business reality: one customer places many orders, one product appears in many order lines, one date has many orders, and one location hosts many orders — but each order line belongs to exactly one customer, one product, one date, and one location. This is the standard and correct cardinality for a star schema and was verified as "One" on the dimension side and "Many" on the fact side for every relationship in Model view.

### Filter direction decisions

All relationships use **single-direction** cross-filtering (dimension filters fact, not the reverse). This was chosen because there are no legitimate many-to-many relationships anywhere in this model, and bidirectional filtering was not needed for any analytical requirement — using it regardless would risk ambiguous filter paths without providing any analytical benefit. Single-direction filtering keeps the model's filter propagation predictable and easy to reason about.

### Modelling challenges encountered

1. **No natural key for location.** The source data has no single column uniquely identifying a location, so a composite `LocationKey` (Country|State|City) was engineered in Power Query and applied consistently to both `DimLocation` and `FactSales`.
   
3. **Reference-query dependency cascade.** Initially, trimming descriptive columns directly out of the main query broke the dimension queries that referenced the same source, because Power Query reference queries evaluate against the live, final output of their source rather than a snapshot taken at creation time. This was resolved by introducing an intermediate `Staging` query (load disabled) holding the full cleaned column set, with `FactSales` and all four dimension tables built as independent references from `Staging` rather than from each other — decoupling them so any one table could be trimmed without breaking the others.
   
5. **Two competing ship-date relationships.** `DimDate[Date]` could theoretically relate to either `FactSales[OrderDate]` or `FactSales[ShipDate]`, but Power BI only permits one active relationship between two tables. The relationship to `OrderDate` was kept active (since order-date-based analysis was the primary requirement); a relationship to `ShipDate` was not created, since it wasn't required for this project's core measures.



<img width="1919" height="1074" alt="03_model" src="https://github.com/user-attachments/assets/8dfa0cd2-aaa1-4fe0-b61d-e3f5796d00c4" />



## 4. DAX Measures & Business Calculations

A dedicated `Measures` table was created to hold all DAX measures separately from any data table, for organizational clarity.

### Six most important measures — detailed explanation

**1. Total Sales**
```dax
Total Sales = SUM(FactSales[Sales])
```
*What it calculates:* The sum of all sales revenue in the current filter context.
*Why useful:* The foundational KPI referenced by nearly every other measure and every dashboard page.
*Main DAX functions:* `SUM`.
*Filter context:* Recalculates automatically within whatever slicers/filters are active (e.g., a specific Year, Region, or Category).
*Used in:* Page 1 KPI cards; denominator in Profit Margin %, Category Contribution %, and other ratio measures throughout.

**2. Profit Margin %**
```dax
Profit Margin % = DIVIDE([Total Profit], [Total Sales], 0)
```
*What it calculates:* Profit as a percentage of sales.
*Why useful:* Sales volume alone can mask an unprofitable business; margin is the metric that actually reflects health.
*Main DAX functions:* `DIVIDE` (with a zero-fallback to avoid division errors when Sales is 0 in a given filter context).
*Filter context:* Changes as slicers filter the underlying transactions — e.g., filtering to `DiscountBand = "High (50%+)"` returns a large negative margin.
*Used in:* Page 1 KPI card; central to the discount-band matrix on Page 3.

**3. YoY Sales Growth %**
```dax
Previous Year Sales = CALCULATE([Total Sales], SAMEPERIODLASTYEAR(DimDate[Date]))
YoY Sales Growth % = DIVIDE([Total Sales] - [Previous Year Sales], [Previous Year Sales], 0)
```
*What it calculates:* Percentage change in sales versus the same period one year earlier.
*Why useful:* Distinguishes genuine growth from a single strong year, and reveals whether momentum is accelerating or slowing.
*Main DAX functions:* `CALCULATE`, `SAMEPERIODLASTYEAR` (time intelligence), `DIVIDE`.
*Filter context:* `SAMEPERIODLASTYEAR` shifts the date filter context back one year while keeping all other active filters (Region, Category, etc.) intact, so growth can be examined for any sliced subset of the business.
*Used in:* Page 1 trend chart.

**4. Product Rank by Profit**
```dax
Product Rank by Profit = RANKX(ALL(DimProduct[ProductName]), [Total Profit], , DESC)
```
*What it calculates:* Each product's rank by total profit, from highest (1) to lowest.
*Why useful:* Directly answers "which products matter most," feeding the Top-10 table on Page 2.
*Main DAX functions:* `RANKX`, `ALL` (to rank against the full unfiltered product list regardless of any single product's own row filter).
*Filter context:* `ALL(DimProduct[ProductName])` removes any existing filter on product name so ranking happens across the complete product list, while other filters (e.g. Category, Region) still apply if active.
*Used in:* Page 2 Top-10 products table.

**5. Loss Order %**
```dax
Loss-Making Orders = CALCULATE(COUNTROWS(FactSales), FactSales[ProfitFlag] = "Loss")
Loss Order % = DIVIDE([Loss-Making Orders], [Total Transactions], 0)
```
*What it calculates:* The percentage of order line items that lose money.
*Why useful:* The core diagnostic metric — reveals how widespread unprofitability is within any sliced subset (region, discount band, etc.), not just its total dollar impact.
*Main DAX functions:* `CALCULATE`, `COUNTROWS`, `DIVIDE`.
*Filter context:* Recalculates per whatever dimension a visual is broken out by — e.g., by `Region` in a bar chart, giving a per-region loss rate.
*Used in:* Page 3 diagnostic matrix and regional bar chart — the basis of the report's central finding that discounting drives losses.

**6. Category Contribution %**
```dax
Category Contribution % =
VAR CategorySales = [Total Sales]
VAR TotalSalesAll = CALCULATE([Total Sales], ALL(DimProduct[Category]))
RETURN DIVIDE(CategorySales, TotalSalesAll, 0)
```
*What it calculates:* Each category's share of total sales within the current filter context.

*Why useful:* Shows relative contribution without needing a separate "grand total" visual.

*Main DAX functions:* `VAR`/`RETURN`, `CALCULATE`, `ALL`, `DIVIDE`.

*Filter context:* `ALL(DimProduct[Category])` removes only the Category filter, so if a Year or Region slicer is also active, the denominator respects that slicer while still summing across all categories — giving contribution-within-context rather than contribution against the unfiltered grand total.

*Used in:* Page 2 category breakdown visuals.

*(Additional measures — `Total Profit`, `Total Transactions`, `Average Order Value`, `Distinct Customers`, `Avg Shipping Duration`, `High Discount Sales`, `Shipping Speed Segment`, and others — are implemented in the model; see the Measures table in the .pbix file for the complete list of 12+ measures.)*


## 5. Dashboard Design

The report contains three pages, structured to move from overview to diagnosis:

**Page 1 — Executive Overview.** KPI cards (Total Sales, Total Profit, Profit Margin %, Total Transactions), a Sales & Profit trend chart by month with drill-down to Year/Quarter/Month, a filled map of Sales by Country, a Sales by Category bar chart, and Year/Market/Segment slicers. A dynamic title reflects the currently selected year.

**Page 2 — Product Analysis.** A Profit by Sub-Category bar chart (conditionally colour-coded red for losses), a Discount vs Profit Margin % scatter chart sized by Sales, a Top-10 Products by Profit table, Category/DiscountBand slicers, and a Category → Sub-Category → Product drill-down hierarchy.

**Page 3 — Diagnostic Analysis.** A ProfitFlag × DiscountBand matrix (the report's central diagnostic finding), a Loss Order % by Region bar chart, an Average Shipping Duration by Ship Mode/Order Priority column chart, a bookmark-driven toggle between "By Region" and "By Segment" views, and a drillthrough page showing full order history for any selected product.

<img width="1477" height="748" alt="04_dashboard_overview1" src="https://github.com/user-attachments/assets/9b2c1496-afaa-4ad1-879f-ef8e08d82f5a" />
<img width="1294" height="721" alt="05_dashboard_analysis" src="https://github.com/user-attachments/assets/aab702a9-be79-4416-b3ba-98efa389c3cc" />
<img width="1219" height="728" alt="06_dashboard_insights" src="https://github.com/user-attachments/assets/9d1c17ff-9ca8-400f-8dfb-0692f4f1b7ee" />



## 6. Key Analytical Results

- **Discounting is the dominant driver of unprofitability.** Every order discounted above 50% loses money (100% loss rate); orders discounted 20–50% lose money 84% of the time; undiscounted orders never lose money.
- **Only 1 of 17 sub-categories is loss-making overall** — Tables, at a -8.5% margin — while every other sub-category is net profitable.
- **Sales nearly doubled from 2011 to 2014** ($2.26M → $4.30M), but 2014 was the first year profit growth (23.9%) trailed sales growth (26.3%), a signal that recent growth may be increasingly discount-driven.
- **Southeast Asia (53.3%) and EMEA (31.9%) have the highest loss-order rates**, well above the 22% cross-region average, while Canada and North Asia post the lowest (0% and 9.5% respectively).
- **Canada, despite low sales volume, has the strongest margin (26.6%) and a 0% loss rate**, suggesting its discounting discipline could be studied as a benchmark for weaker-performing markets.

*(Full supporting charts and detail: see `Superstore_BI_Report.docx` in this repository, or the corresponding visuals on Power BI Pages 2–3.)*



## 7. Repository Structure

```
DSA3050-PowerBI-YourName-RegNo/
│
├── README.md
│
├── data/
│   └── dataset.csv
│
├── powerbi/
│   └── DSA3050_YourName.pbix
│
└── screenshots/
    ├── 01_raw_data.png
    ├── 02_power_query.png
    ├── 03_model.png
    ├── 04_dashboard_overview.png
    ├── 05_dashboard_analysis.png
    └── 06_dashboard_insights.png
```
