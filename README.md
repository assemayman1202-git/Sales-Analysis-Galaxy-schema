# 🌌 Sales Performance — Galaxy Schema Analytics

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-217346?style=for-the-badge&logo=microsoftexcel&logoColor=white)
![Data Modeling](https://img.shields.io/badge/Data%20Modeling-Galaxy%20Schema-2E86C1?style=for-the-badge&logo=databricks&logoColor=white)
![Row Level Security](https://img.shields.io/badge/Row%20Level%20Security-Enabled-C0392B?style=for-the-badge&logo=shieldsdotio&logoColor=white)

---

## 📌 Project Overview

An end-to-end Power BI analytics project built on a **Galaxy Schema (Fact Constellation)** — five fact tables sharing a set of conformed dimensions across Sales, Campaigns, Inventory, and Order Fulfillment. The report is decomposed into **4 focused pages** (Overview, Campaigns, Inventories, Orders), each with its own KPI set, DAX measures, and slicers, plus **Row-Level Security (RLS)** enforced by region so every user only sees the data relevant to them.

| Metric | Value |
|---|---|
| Actual Revenue | 525.96K |
| Revenue YoY % | 0.98 |
| Avg Order Value | 6.57K |
| Gross Margin % | 0.37 |
| Total Campaign Spend | 78.84K |
| Total Clicks | 224K |
| Inventory Value | 74K |
| Inventory Turnover Ratio | 53.86 |
| Avg Ship Lead Time | 2.75 days |
| Order to Cash Cycle | 32.84 days |

---

## 🗂️ Repository Structure

```
Galaxy-Schema-Sales-Project/
│
├── Galaxy_Schema_Sales_Project.pbix     # Power BI report file
│
├── galaxy modelling.png                  # Data model diagram (galaxy schema)
├── galaxy overview.png                   # Overview page screenshot
├── galaxy campaigns.png                  # Campaigns page screenshot
├── galaxy Inventories.png                # Inventories page screenshot
├── galaxy orders.png                     # Orders page screenshot
├── RLS security.png                      # Manage Security Roles screenshot
├── RLS users.png                         # View as Roles test screenshot
│
└── README.md
```

---

## 🏗️ Architecture & Data Model — Galaxy Schema (Fact Constellation)

![galaxy_modelling](modelling/galaxy%20modelling.png)

Unlike a single star schema, this model is a true **Galaxy Schema**: multiple fact tables at different grains, all sharing conformed dimensions, so the same `dim_product`, `dim_customer`, `dim_geo`, and `Dim_Date` can answer questions across sales, marketing, inventory, and fulfillment without duplicating logic.

| Table | Type | Key Fields |
|---|---|---|
| `fact_sales` | Fact | OrderID, LineID, product_key, CustomerID, geo_key_bill, geo_key_ship, flag_key, Quantity, UnitCost, LineTotal, DiscountPct |
| `fact_campaign` | Fact | campaign_key, Date, Clicks, Impressions, Spend |
| `fact_sales_targets` | Fact | month, TargetRevenue |
| `fact_Inventory` | Fact | month, product_key, Value |
| `fact_order_fulfillment` | Fact | OrderID, CustomerID, OrderDate, ShipDate, DeliveryDate, InvoiceID, InvoiceDate, PayDate |
| `factless fact` | Bridge (factless) | campaign_key, product_key — links campaigns to the products they promoted |
| `dim_product` | Dimension | product_key, ProductName, ProductCode, Brand, Category, Subcategory, PrimarySupplier, UnitPrice |
| `dim_customer` | Dimension | CustomerID, CustomerName, AccountManager, Segment, RegionName, CityName, CreditLimit, PaymentTerms |
| `dim_geo` | Dimension | geo_key, CityName, RegionName |
| `dim_campaign` | Dimension | campaign_key, CampaignName, Channel, Budget, StartDate, EndDate |
| `dim_order_flag` (junk dim) | Dimension | flag_key, OrderChannel, channel name, Priority, Status |
| `Dim_Date` | Dimension | Date, Day, Day Name, Month Name, Month Number, Quarter, Year |
| `security` | RLS Table | Region, UserEmail |
| `_measures` | Measures | All DAX measures — Actual Revenue, Avg Order Value, ROAS, Inventory Turnover, DSO, etc. |

### Pipeline / Build Steps

1. **Ingestion** — Loaded all five fact tables and their dimensions into Power BI, keeping each fact at its natural grain (line-level sales, daily campaign performance, monthly inventory snapshots, per-order fulfillment events).
2. **Calendar Modeling** — Built a single `Dim_Date` table covering the full date range across **every** fact table (not just `fact_sales`), preventing blank-slicer issues caused by dates falling outside a narrower range. Standardized every date column to the same data type across all facts to guarantee relationship matches.
3. **Relationship Design** — Connected each fact to its dimensions with single-direction, one-to-many relationships. Where a fact needed more than one date or geography role (e.g. `fact_sales[geo_key_bill]` vs `geo_key_ship`, or `fact_order_fulfillment`'s five date columns), kept **one relationship Active** and the rest **Inactive**, resolved in DAX with `USERELATIONSHIP` instead of allowing ambiguous multi-path relationships.
4. **Avoiding Ambiguous Paths** — Deliberately did **not** create a physical relationship between `dim_campaign` and `Dim_Date` (via StartDate/EndDate) in addition to the existing `fact_campaign → Dim_Date` path, since that would create two paths between the same tables. Campaign-period logic was instead handled with a `FILTER`-based DAX measure.
5. **Factless Fact for Attribution** — Used `factless fact` (campaign_key, product_key) purely as a bridge table with no measure column, enabling "which products were promoted by which campaign" filtering via `TREATAS` rather than a risky bidirectional relationship across the whole model.
6. **DAX Layer** — Built all KPIs into a dedicated `_measures` table: revenue and margin measures, campaign efficiency (CTR, CPC, ROAS), inventory health (turnover, days on hand), and fulfillment cycle times (ship lead time, DSO, order-to-cash).
7. **Row-Level Security** — Implemented dynamic RLS scoped by region (details below).
8. **Report Build** — Delivered a 4-page interactive report (Overview, Campaigns, Inventories, Orders), each with page-specific slicers, synced where relevant (Date, Region) so filtering flows consistently across the whole report.

---

## 🔐 Row-Level Security (RLS)

RLS was implemented so each user only sees the region(s) they're authorized for, without maintaining a separate role per region.

![RLS_security](Row%20level%20security/RLS%20security.png)

A single role, **"regional access"**, was created and applied to both `dim_customer` and `dim_geo` (since region information exists on both tables and both feed into `fact_sales`), using a dynamic lookup against a dedicated `security` table:

```dax
[RegionName] = LOOKUPVALUE(security[Region], security[UserEmail], USERPRINCIPALNAME())
```

`USERPRINCIPALNAME()` returns the signed-in user's email at query time, so the same rule automatically resolves to a different region per user — no need to hard-code or duplicate roles.

![RLS users](Row level security (RLS)/RLS%20users.png)

Testing was done using **View as roles → regional access + Other user**, entering a real email from the `security` table (e.g. `omar.farouk@arka.com`) to simulate that user's session — confirming the model correctly restricted every visual to that user's assigned region only, across all four report pages.

---

## 📊 Power BI Dashboard

### Page 1 — Overview

![galaxy_overview](Dashboards/galaxy%20overview.png)

**KPIs:** Actual Revenue (525.96K) · Revenue YoY % (0.98) · Avg Order Value (6.57K) · Gross Margin % (0.37)

**Visuals:**
- 📊 Bar Chart: Top 10 Products by Revenue — Team M047 leads (32K), followed by Phones M036 (28K), Skincare M012 (22K), Audio M020 (22K), Audio M002 (21K)
- 📈 Line Chart: Sales Trend over time — visible seasonal peaks (Jul 2025, Jan 2026) and troughs, ranging roughly between 6K–47K per period
- 🍩 Donut Chart: Sales by Channel — Wholesale (143.69K, 27.32%) and Retail Partner (142.08K, 27.01%) lead, followed by Field Sale (125.97K, 23.95%) and Online Store (114.21K, 21.72%)
- 🎯 Gauge: Sales Performance vs Target — 525.96K achieved against a 552K target
- 📊 Bar Chart: Sales by Category — Electronics leads (0.13M), followed by Apparel (0.10M), Home and Sports (0.08M each), Beauty and Industrial (0.07M each)

**Slicers:** Category · OrderChannel · Date · Region

---

### Page 2 — Campaigns

![galaxy_campaigns](Dashboards/galaxy%20campaigns.png)

**KPIs:** Total Spend (78.84K) · Total Clicks (224K) · Total Impressions (7M) · CPC (0.35)

**Visuals:**
- 📈 Combo Line Chart: CTR & CPC over months — both efficiency metrics tracked monthly to catch performance drift
- 📊 Funnel: Spend by Channel — Paid Search leads (28.07K), then Social (26.75K), Email (14.81K), and Display (9.21K)
- 📋 Table: Campaign Budget Utilization — every campaign tracked against budget, with utilization ranging from 0.94 (Black Friday, under budget) to 1.03 (New Year Clearance, over budget)
- 📋 Table: Campaign Spend by Date Range — Black Friday (Nov 2025, 28.07K) was the highest-spend campaign, followed by Summer Sale (14.61K) and Spring Launch 2026 (12.14K); total spend across all 6 campaigns: 78.84K against an 81K combined budget (97% overall utilization)

**Slicers:** Channel · CampaignName · Date

---

### Page 3 — Inventories

![galaxy_Inventories](Dashboards/galaxy%20Inventories.png)

**KPIs:** Inventory Value (74K) · Inventory Turnover Ratio (53.86) · Days of Inventory On Hand (6.78) · Inventory to Sales Ratio (0.14)

**Visuals:**
- 📊 Bar Chart: Best Product Supplier — Supplier A leads (7.0K), closely followed by Supplier E (6.9K) and Supplier B (6.7K); Supplier M trails significantly (3.3K)
- 📈 Line Chart: Inventory Value Trend — fluctuating monthly between 4.7K and 7.8K across 2025, with a notable dip in May 2025 (5.0K) and a peak in July 2025 (7.8K)
- 🌳 Treemap: Inventory Value by Category & Brand — Apparel (18K) and Electronics (16K) hold the most value, followed by Home (12K), Beauty and Industrial (10K each), and Sports (8K)

**Slicers:** Category · Brand · Date · Subcategory

---

### Page 4 — Orders

![galaxy_orders](Dashboards/galaxy%20orders.png)

**KPIs:** Avg Ship Lead Time (2.75 days) · Avg Delivery Time (5.72 days) · Order to Cash Cycle (32.84 days) · On-Time Delivery % (1.00)

**Visuals:**
- 🍩 Donut Chart: Sales by Payment Terms — Prepaid dominates (83.22K, 38.98%), followed by Net 60 (70.61K, 33.07%), Net 15 (32.91K, 15.4%), and Net 30 (26.75K, 12.53%)
- 🍩 Donut Chart: Sales by Segment — Mid-Market leads (79.37K, 37.18%), followed by Enterprise (73.08K, 34.23%) and SMB (61.04K, 28.59%)
- 🔻 Funnel: Ship to City — Istanbul at the top of shipment volume, tapering down through Singapore, Paris, Cairo, London, Dubai, and 15+ other cities down to Mumbai
- 📊 Bar Chart: Sales by Region — Europe leads (59K), followed by Middle East (50K), Asia Pacific (43K), North America (42K), and Latin America (19K)

**Slicers:** channel name · Segment · Date · Region

---

## 📈 Key Insights & Analysis

### 1. Revenue Is Tracking Close to Target, But Not There Yet
Actual Revenue (525.96K) sits just under the 552K target — a **95% attainment rate**. The Revenue YoY % of 0.98 confirms growth is nearly flat year-over-year rather than accelerating, meaning current momentum alone won't close the gap without a deliberate push in underperforming categories.

### 2. Wholesale and Retail Partner Channels Are Carrying Revenue
Together, Wholesale (27.32%) and Retail Partner (27.01%) channels generate over half of total revenue, while the Online Store lags at 21.72% — a sizeable gap that suggests the direct-to-consumer channel is underinvested relative to its potential.

### 3. Campaign Spend Efficiency Varies Sharply by Channel
Paid Search and Social together absorb over 69% of the 78.84K campaign budget, but with a blended CPC of 0.35, efficiency should be assessed per-channel rather than in aggregate — Display's lower spend (9.21K) may reflect either intentional deprioritization or an underexplored channel.

### 4. Budget Discipline Is Mostly Solid, With One Overspend
Five of six campaigns landed within ±5% of budget. New Year Clearance is the exception at 103% utilization — a modest overspend worth flagging before the next planning cycle, though not yet a material risk.

### 5. Inventory Turnover Is High, Which Cuts Both Ways
A turnover ratio of 53.86 and just 6.78 days of inventory on hand indicates very fast-moving stock — efficient from a holding-cost perspective, but it also raises stockout risk if replenishment lead times can't keep pace with sales velocity.

### 6. Apparel and Electronics Concentrate the Most Inventory Value
These two categories together hold roughly 46% of total inventory value (34K of 74K). Any supply disruption or demand shift in either category would have an outsized effect on overall inventory health.

### 7. Order Fulfillment Is Fast, But the Cash Cycle Is the Real Bottleneck
Ship Lead Time (2.75 days) and Delivery Time (5.72 days) are both tight, but the full Order-to-Cash Cycle stretches to 32.84 days — meaning the gap between delivery and payment collection (driven by Net 30/Net 60 terms) is where the real cash-flow lag lives, not the logistics side.

### 8. Payment Terms Skew Toward Extended Credit
Nearly half of revenue (Net 60 + Net 15 + Net 30 combined ≈ 61%) is collected on extended terms rather than Prepaid (38.98%) — a meaningful portion of revenue is effectively financed to customers, reinforcing why the cash cycle runs longer than the fulfillment cycle alone would suggest.

### 9. Europe Leads Regional Sales, Latin America Lags Significantly
Europe (59K) outsells Latin America (19K) by roughly 3×. Combined with the newly implemented RLS by region, this creates a natural framework for regional managers to own and act on their own performance gaps without needing visibility into others' data.

---

## 🎯 Recommendations

| Area | Recommendation | Expected Impact |
|---|---|---|
| Revenue vs Target | Investigate underperforming categories (Beauty, Industrial) and reallocate promotional focus toward Electronics/Apparel, which already lead | Close the ~5% gap to target |
| Online Store Channel | Increase paid and organic investment in the Online Store channel given its lag versus Wholesale/Retail Partner | Rebalance channel mix, reduce over-reliance on partners |
| Campaign Budget | Set a hard budget alert threshold (e.g. 100%) for campaigns like New Year Clearance that historically overspend | Prevent recurring overspend |
| Inventory Risk | Set minimum stock thresholds for Apparel and Electronics given their high value concentration and fast turnover | Reduce stockout risk on highest-value categories |
| Cash Cycle | Review Net 60 terms for lower-priority segments; consider incentivizing Prepaid or Net 15 for new accounts | Shorten the 32.84-day Order-to-Cash Cycle |
| Regional Focus | Use the new RLS setup to give Latin America's regional manager a dedicated, filtered view to drive a targeted growth plan | Close the regional performance gap |
| Data Governance | Extend RLS coverage to `fact_campaign` and `fact_Inventory` if regional segmentation becomes relevant to marketing/inventory teams | Consistent security posture across the whole model |

---

## 🛠️ Tech Stack

<p>
<img src="https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black" alt="Power BI"/>
<img src="https://img.shields.io/badge/DAX-217346?style=for-the-badge&logo=microsoftexcel&logoColor=white" alt="DAX"/>
<img src="https://img.shields.io/badge/Data%20Modeling-Galaxy%20Schema-2E86C1?style=for-the-badge" alt="Data Modeling"/>
<img src="https://img.shields.io/badge/Power%20Query-742774?style=for-the-badge" alt="Power Query"/>
</p>

| Tool | Role |
|---|---|
| **Power BI Desktop** | Data modeling, RLS configuration, 4-page interactive dashboard |
| **Power Query** | Data cleaning, date type standardization across all fact tables |
| **DAX** | All KPI measures — Revenue, Margin, ROAS, Inventory Turnover, DSO, Order-to-Cash Cycle, and the RLS filter expressions |
| **Galaxy Schema (Fact Constellation)** | Data model — 5 fact tables, 1 factless bridge fact, 7 dimension tables (including a junk dim and an RLS table), 1 measures table |
| **Row-Level Security (RLS)** | Dynamic region-based access control via `LOOKUPVALUE` + `USERPRINCIPALNAME()` |

---

## 🚀 How to Reproduce

1. Open Power BI Desktop → `Get Data` → load the five fact tables and dimension tables
2. In Power Query, standardize all date columns to the same data type across every fact table
3. Build a single `Dim_Date` calendar table covering the full date range across all facts (not just one)
4. Model the relationships: one active relationship per fact-to-dimension pair; mark secondary date/geo relationships as inactive and resolve them with `USERELATIONSHIP` in DAX
5. Add the `factless fact` bridge table between `dim_campaign` and `dim_product` — do **not** create a direct relationship between `dim_campaign` and `Dim_Date` if one already exists via `fact_campaign`, to avoid an ambiguous path
6. Build all DAX measures into the `_measures` table (Revenue, Margin, CTR/CPC/ROAS, Inventory Turnover, DSO, Order-to-Cash Cycle, etc.)
7. Create the `security` table (Region, UserEmail) and configure RLS: `Modeling → Manage Roles → New Role → LOOKUPVALUE(security[Region], security[UserEmail], USERPRINCIPALNAME())` applied to `dim_customer` and `dim_geo`
8. Test with `Modeling → View as roles → [role] + Other user + a real email from the security table`
9. Build the 4 report pages (Overview, Campaigns, Inventories, Orders) with page-specific slicers, syncing Date and Region slicers across pages
10. Or open the `.pbix` file directly

---

## 👤 Author

Built as a personal Power BI portfolio project demonstrating galaxy schema (fact constellation) design, multi-fact DAX modeling, and dynamic Row-Level Security implementation for a multi-department sales analytics use case.

---

## 📄 License

This project is licensed under the MIT License.
