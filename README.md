# Customer Segmentation & LTV Dashboard — Tableau Build Guide

## Package contents
| File | Purpose |
|---|---|
| `orders.csv` | 19,960 synthetic order lines, 2,500 customers, Jan 2023 – Sep 2026 |
| `Customer_Segmentation_LTV.twb` | Workbook with data source, 6 parameters and ~28 calculated fields pre-defined |
| `calculated_fields.txt` | Every formula as plain text (fallback if the workbook needs fields re-created) |
| `README.md` | This guide |

## 1. Open the workbook
1. Unzip the folder (keep `orders.csv` next to the `.twb`).
2. Open `Customer_Segmentation_LTV.twb` in Tableau Desktop (2022.1+).
3. If Tableau says it can't find the file, click **Edit Data Source -> Locate** and point to `orders.csv`.
4. If any calculated field shows an error, copy its formula from `calculated_fields.txt`. Create parameters first, then fields in listed order.

The workbook ships with no sheets. The build below takes about 30-40 minutes.

## 2. What's pre-built

**Parameters (right-click -> Show Parameter on the dashboard):**
| Parameter | Default | Drives |
|---|---|---|
| Recency Step (days) | 90 | R score bands (<=90d = 5, <=180d = 4, ...) |
| Frequency Step (orders) | 2 | F score bands |
| Monetary Step ($) | 300 | M score bands |
| High Value CLV Threshold ($) | 1000 | `Is High Value` flag / "High Value" definition |
| Churn Window (days) | 180 | `Is Churned` flag |
| Expected Lifespan (years) | 3.0 | Predicted CLV |

**LOD expressions**
- `FIXED [Customer ID]` -> Recency, Frequency, Monetary, Profit, first/last order. Customer-level metrics stay constant whatever dimensions are in the view.
- `FIXED` (no dimension) -> Snapshot Date (dataset max order date).
- `INCLUDE [Customer ID]` -> `Avg CLV per Customer (INCLUDE)` and `Avg Sales per Customer (INCLUDE)`. Per-customer values are computed first, then averaged at whatever grain the view has (segment, region, channel).

**CLV model**
- Historic CLV = lifetime profit to date.
- Predicted CLV = Avg Order Value x Orders per Year x Margin % x Lifespan. Churned customers are discounted to 25%.
- Total CLV = Historic + Predicted.

## 3. Build the sheets

**Sheet 1 – KPI tiles.** `Customer Count`, `SUM(Sales)`, `Avg CLV per Customer (INCLUDE)`, `Churn Rate (customers)`. One text sheet each, or one Measure Names/Values sheet.

**Sheet 2 – RFM segment overview.** Treemap or bar: `RFM Segment` on Color/Label, `Customer Count` on Size. Add `Avg CLV per Customer (INCLUDE)` to Tooltip.

**Sheet 3 – RFM heat map.** Columns: `F Score`, Rows: `R Score`, Color: `Avg CLV per Customer (INCLUDE)`, Label: `Customer Count`. Add `M Score` as a quick filter.

**Sheet 4 – K-Means clustering.**
1. Drag `Customer ID` to **Detail**; set mark type to **Circle**.
2. Columns: `AVG(Frequency (orders))`, Rows: `AVG(Monetary (sales))`. Drag `AVG(Recency (days))` to Size.
   - Use AVG because the FIXED fields are constant per customer.
3. **Analytics pane -> Cluster** onto the view. In the dialog: Variables = Recency, Frequency, Monetary only; **Number of Clusters = 4** (or leave Automatic and check the Describe Clusters summary).
4. Rename the cluster pill. Drag it to **Data pane** to save as a group field named `ML Cluster`.
5. Right-click `ML Cluster` -> **Describe Clusters** to name each cluster after its centroid (e.g. "Big Spenders", "Dormant").

Tip: Compare `ML Cluster` against `RFM Segment` in a crosstab. It shows where rule-based and ML segmentation agree.

**Sheet 5 – CLV by segment/channel.** Rows: `Acquisition Channel`, Columns: `Avg CLV per Customer (INCLUDE)`, Color: `Value Tier` or `RFM Segment`. This is what to use for marketing budget allocation.

**Sheet 6 – Churn risk.** Bar of `Churn Rate (customers)` by `RFM Segment` or `Region`. Reference line at overall rate.

**Sheet 7 – Customer list (action target).** Text table: `Customer ID`, `Customer Name`, `RFM Segment`, `RFM Code`, `Recency (days)`, `Total CLV`. Filter: `Is High Value = True` and `Is Churned = True` produces your win-back list.

## 4. Sets (High Value definition)
1. Right-click `Customer ID` -> **Create -> Set...** -> name it `High Value Customers`.
2. **Condition** tab -> By field: `Total CLV` -> **Maximum** -> `>=` and enter a value, or use **Top** tab -> Top 20% by `Total CLV`.
3. Keep `Is High Value` (parameter-driven) for the dynamic definition, and the Set for fixed Top-N behaviour. A Set can also be IN/OUT-compared against its complement on any sheet.
4. Optional: create a **Combined Set** of `High Value Customers` and customers where `Is Churned` = True for the at-risk high-value cohort.

## 5. Dashboard
Size: Automatic (or 1400 x 900).
- Top row: parameter controls (all 6) and KPI tiles.
- Middle: Segment overview (left), RFM heat map (center), Cluster scatter (right).
- Bottom: CLV by channel, churn by segment, customer list.
- **Actions:** Dashboard -> Actions -> Add **Filter** (Select, source = Sheets 2, 3, 4, target = all), and **Highlight** on `RFM Segment`.
- Add quick filters for Region, Category, Acquisition Channel, Order Date (set the filter on Order Date to context if you filter by year, otherwise FIXED LODs ignore it).

## 6. Important Tableau behaviour to know
- **FIXED LODs ignore dimension filters** unless the filter is added to context. RFM fields are customer-lifetime by design.
- Clustering requires the pill structure in step 4 (detail at customer level). Clusters are recomputed on filter change.
- The `Predicted CLV` is a simplified deterministic model. For production use, replace it with BG/NBD + Gamma-Gamma outputs from Python/R and join them on `Customer ID`.
- Data is synthetic. Replace `orders.csv` with your own (same columns) and the dashboard continues to work.

---

## Appendix: Formula reference
```
PARAMETERS
- Recency Step (days) | integer | default 90 | range 30-180 step 15
- Frequency Step (orders) | integer | default 2 | range 1-5 step 1
- Monetary Step ($) | integer | default 300 | range 100-1000 step 50
- High Value CLV Threshold ($) | integer | default 1000 | range 500-5000 step 100
- Churn Window (days) | integer | default 180 | range 60-365 step 30
- Expected Lifespan (years) | real | default 3.0 | range 1-10 step 0.5

CALCULATED FIELDS (create in this order)

// Snapshot Date
{ FIXED : MAX([Order Date]) }

// Customer First Order
{ FIXED [Customer ID] : MIN([Order Date]) }

// Customer Last Order
{ FIXED [Customer ID] : MAX([Order Date]) }

// Recency (days)
DATEDIFF('day',[Customer Last Order],[Snapshot Date])

// Frequency (orders)
{ FIXED [Customer ID] : COUNTD([Order ID]) }

// Monetary (sales)
{ FIXED [Customer ID] : SUM([Sales]) }

// Customer Profit
{ FIXED [Customer ID] : SUM([Profit]) }

// R Score
MAX(1,6-CEILING(MAX(1,[Recency (days)])/[Recency Step (days)]))

// F Score
MIN(5,CEILING([Frequency (orders)]/[Frequency Step (orders)]))

// M Score
MIN(5,CEILING([Monetary (sales)]/[Monetary Step ($)]))

// RFM Score
[R Score]+[F Score]+[M Score]

// RFM Code
STR([R Score])+STR([F Score])+STR([M Score])

// RFM Segment
IF [R Score]>=4 AND [F Score]>=4 AND [M Score]>=4 THEN "Champions"
ELSEIF [R Score]>=3 AND [F Score]>=3 THEN "Loyal Customers"
ELSEIF [R Score]>=4 AND [F Score]<=2 THEN "New / Promising"
ELSEIF [R Score]<=2 AND [F Score]>=3 AND [M Score]>=3 THEN "At Risk - High Value"
ELSEIF [R Score]<=2 AND [F Score]>=3 THEN "At Risk"
ELSEIF [R Score]<=2 THEN "Hibernating / Lost"
ELSE "Needs Attention" END

// Tenure (years)
MAX(DATEDIFF('day',[Customer First Order],[Snapshot Date])/365.0, 1.0/12)

// Avg Order Value
[Monetary (sales)]/[Frequency (orders)]

// Orders per Year
[Frequency (orders)]/[Tenure (years)]

// Margin %
IF [Monetary (sales)]>0 THEN [Customer Profit]/[Monetary (sales)] ELSE 0 END

// Historic CLV
[Customer Profit]

// Predicted CLV
[Avg Order Value]*[Orders per Year]*[Margin %]*[Expected Lifespan (years)]*IF [Is Churned] THEN 0.25 ELSE 1 END

// Total CLV
[Historic CLV]+[Predicted CLV]

// Is Churned
[Recency (days)]>[Churn Window (days)]

// Is High Value
[Total CLV]>=[High Value CLV Threshold ($)]

// Value Tier
IF [Is High Value] THEN "High Value" ELSE "Standard" END

// Customer Count
COUNTD([Customer ID])

// Churn Rate (customers)
COUNTD(IF [Is Churned] THEN [Customer ID] END)/COUNTD([Customer ID])

// Avg CLV per Customer (INCLUDE)
AVG({ INCLUDE [Customer ID] : MAX([Total CLV]) })

// Avg Sales per Customer (INCLUDE)
AVG({ INCLUDE [Customer ID] : SUM([Sales]) })

```
