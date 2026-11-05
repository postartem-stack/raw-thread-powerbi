# RAW THREAD — Power BI Model & DAX Library

Everything needed to build the advanced report: model setup, 60+ measures
grouped by theme, calculated tables, and the patterns that separate a senior
model from a beginner one.

Paste measures into a dedicated **`_Measures`** table (Home → Enter data →
create empty table named `_Measures`, delete its column, then add measures to
it). This keeps the field list clean.

---

## 1. Model setup (do this first — everything depends on it)

### Load
Get Data → Text/CSV → load all nine files. Keep the names as-is.

### Relationships (all one-to-many, single direction, dimension → fact)

| From (1) | To (many) | Key |
|---|---|---|
| DimDate | FactSales | DateKey |
| DimProduct | FactSales | ProductKey |
| DimCustomer | FactSales | CustomerKey |
| DimChannel | FactSales | ChannelKey |
| DimGeography | FactSales | GeographyKey |
| DimDate | FactInventory | DateKey |
| DimProduct | FactInventory | ProductKey |
| DimDate | FactMarketingSpend | DateKey |
| DimChannel | FactMarketingSpend | ChannelKey |

**FactTargets stays unrelated on purpose.** Its grain (month × category ×
channel) doesn't match FactSales, and forcing a physical relationship would
either fan out or blank at day level. It's handled with `TREATAS` virtual
relationships below — this is the correct pattern and worth explaining in the
case study.

### Mark the date table
Table tools → **Mark as date table** → `DimDate[Date]`. Without this, time
intelligence silently misbehaves.

### Hide from report view
Hide every `*Key` column in both dims and facts (right-click → Hide in report
view). Users should filter by `DimProduct[Category]`, never by a surrogate key.

### Hierarchies
- `DimProduct`: Category → Subcategory → Product
- `DimGeography`: Region → Country → City
- `DimDate`: Year → Quarter → MonthShort

### Sort columns
`DimDate[MonthShort]` → sort by `MonthNo`. `DimDate[MonthName]` → sort by
`MonthNo`. Otherwise months sort alphabetically (Apr, Aug, Dec…).

---

## 2. Core measures

```dax
Total Revenue = SUM ( FactSales[Revenue] )

Returns = SUM ( FactSales[ReturnedAmount] )

Net Revenue = [Total Revenue] - [Returns]

Total Cost = SUM ( FactSales[Cost] )

Gross Profit = [Net Revenue] - [Total Cost]

Margin % = DIVIDE ( [Gross Profit], [Net Revenue] )

Orders = DISTINCTCOUNT ( FactSales[OrderID] )

Units Sold = SUM ( FactSales[Quantity] )

Lines = COUNTROWS ( FactSales )

AOV = DIVIDE ( [Net Revenue], [Orders] )

Units per Order = DIVIDE ( [Units Sold], [Orders] )

Return Rate % = DIVIDE ( [Returns], [Total Revenue] )

Avg Discount % =
DIVIDE (
    SUMX ( FactSales, FactSales[UnitPrice] * FactSales[Quantity] * FactSales[DiscountPct] ),
    SUMX ( FactSales, FactSales[UnitPrice] * FactSales[Quantity] )
)
```

---

## 3. Time intelligence

```dax
Revenue LY = CALCULATE ( [Net Revenue], SAMEPERIODLASTYEAR ( DimDate[Date] ) )

Revenue YoY = [Net Revenue] - [Revenue LY]

Revenue YoY % = DIVIDE ( [Revenue YoY], [Revenue LY] )

Revenue PM = CALCULATE ( [Net Revenue], PREVIOUSMONTH ( DimDate[Date] ) )

Revenue MoM % = DIVIDE ( [Net Revenue] - [Revenue PM], [Revenue PM] )

Revenue YTD = TOTALYTD ( [Net Revenue], DimDate[Date] )

-- fiscal year starts 1 Feb, so it closes 31 Jan
Revenue FYTD = TOTALYTD ( [Net Revenue], DimDate[Date], "01-31" )

-- moving annual total (rolling 12 months)
Revenue MAT =
CALCULATE (
    [Net Revenue],
    DATESINPERIOD ( DimDate[Date], MAX ( DimDate[Date] ), -12, MONTH )
)

Revenue R3M Avg =
AVERAGEX (
    DATESINPERIOD ( DimDate[Date], MAX ( DimDate[Date] ), -3, MONTH ),
    [Net Revenue]
)

-- cumulative within the visible period
Revenue Running Total =
CALCULATE (
    [Net Revenue],
    FILTER ( ALLSELECTED ( DimDate[Date] ), DimDate[Date] <= MAX ( DimDate[Date] ) )
)

Margin % LY = CALCULATE ( [Margin %], SAMEPERIODLASTYEAR ( DimDate[Date] ) )

Margin % pts YoY = [Margin %] - [Margin % LY]
```

> **Why `Margin % pts YoY` and not a percentage change:** margin is already a
> percentage — comparing it in % change misleads. Percentage points is the
> honest unit. Details like this read as senior.

---

## 4. Budget vs actual (TREATAS pattern)

```dax
Target Revenue =
CALCULATE (
    SUM ( FactTargets[TargetRevenue] ),
    TREATAS ( VALUES ( DimDate[YearMonth] ),      FactTargets[YearMonth] ),
    TREATAS ( VALUES ( DimProduct[Category] ),    FactTargets[Category] ),
    TREATAS ( VALUES ( DimChannel[ChannelKey] ),  FactTargets[ChannelKey] )
)

Target Margin =
CALCULATE (
    SUM ( FactTargets[TargetMargin] ),
    TREATAS ( VALUES ( DimDate[YearMonth] ),      FactTargets[YearMonth] ),
    TREATAS ( VALUES ( DimProduct[Category] ),    FactTargets[Category] ),
    TREATAS ( VALUES ( DimChannel[ChannelKey] ),  FactTargets[ChannelKey] )
)

Revenue vs Target = [Net Revenue] - [Target Revenue]

Revenue vs Target % = DIVIDE ( [Revenue vs Target], [Target Revenue] )

Target Attainment % = DIVIDE ( [Net Revenue], [Target Revenue] )

-- traffic-light status for a card or table
Attainment Status =
VAR a = [Target Attainment %]
RETURN
SWITCH (
    TRUE (),
    ISBLANK ( a ), "—",
    a >= 1,    "On target",
    a >= 0.9,  "At risk",
    "Below target"
)
```

**Note the grain:** targets are monthly, so use these at month level or
higher. At day level the target would repeat — guard it if needed:

```dax
Target Revenue (grain-safe) =
IF (
    ISFILTERED ( DimDate[Date] ) && NOT ISFILTERED ( DimDate[YearMonth] ),
    BLANK (),
    [Target Revenue]
)
```

---

## 5. Customer analytics

### Calculated column on FactSales (needed for cohorts)

```dax
Months Since Acquisition =
DATEDIFF (
    RELATED ( DimCustomer[AcquisitionDate] ),
    RELATED ( DimDate[Date] ),
    MONTH
)
```

### Measures

```dax
Active Customers = DISTINCTCOUNT ( FactSales[CustomerKey] )

New Customers =
CALCULATE (
    DISTINCTCOUNT ( DimCustomer[CustomerKey] ),
    USERELATIONSHIP ( DimCustomer[AcquisitionDate], DimDate[Date] )
)
```
> Requires an **inactive** relationship `DimCustomer[AcquisitionDate] →
> DimDate[Date]`. Create it, set it inactive, then activate it inside the
> measure. This is the role-playing-dimension pattern.

```dax
Returning Customers = [Active Customers] - [New Customers]

Cohort Size =
CALCULATE ( DISTINCTCOUNT ( DimCustomer[CustomerKey] ), REMOVEFILTERS ( FactSales ) )

Retention % = DIVIDE ( [Active Customers], [Cohort Size] )

Revenue per Customer = DIVIDE ( [Net Revenue], [Active Customers] )

Orders per Customer = DIVIDE ( [Orders], [Active Customers] )

-- simple lifetime value proxy
Customer LTV = DIVIDE ( [Gross Profit], [Active Customers] )

Repeat Purchase Rate =
VAR RepeatCustomers =
    COUNTROWS (
        FILTER (
            VALUES ( FactSales[CustomerKey] ),
            CALCULATE ( DISTINCTCOUNT ( FactSales[OrderID] ) ) > 1
        )
    )
RETURN DIVIDE ( RepeatCustomers, [Active Customers] )
```

**Cohort matrix visual:** Matrix → Rows `DimCustomer[AcquisitionYearMonth]`,
Columns `FactSales[Months Since Acquisition]`, Values `Retention %`.

### RFM segmentation (calculated table)

```dax
RFM =
VAR LastDate = MAX ( FactSales[DateKey] )
VAR Base =
    ADDCOLUMNS (
        VALUES ( DimCustomer[CustomerKey] ),
        "Recency",   DATEDIFF ( CALCULATE ( MAX ( DimDate[Date] ) ), TODAY (), DAY ),
        "Frequency", CALCULATE ( DISTINCTCOUNT ( FactSales[OrderID] ) ),
        "Monetary",  CALCULATE ( [Net Revenue] )
    )
VAR Scored =
    ADDCOLUMNS (
        Base,
        "R", 6 - ROUNDUP ( PERCENTRANK.INC ( SELECTCOLUMNS ( Base, [Recency] ), [Recency] ) * 5, 0 ),
        "F", ROUNDUP ( PERCENTRANK.INC ( SELECTCOLUMNS ( Base, [Frequency] ), [Frequency] ) * 5, 0 ),
        "M", ROUNDUP ( PERCENTRANK.INC ( SELECTCOLUMNS ( Base, [Monetary] ), [Monetary] ) * 5, 0 )
    )
RETURN
ADDCOLUMNS (
    Scored,
    "RFM Segment",
        SWITCH (
            TRUE (),
            [R] >= 4 && [F] >= 4, "Champions",
            [R] >= 3 && [F] >= 3, "Loyal",
            [R] >= 4 && [F] <= 2, "New / Promising",
            [R] <= 2 && [F] >= 4, "At Risk",
            [R] <= 2 && [F] <= 2, "Lost",
            "Needs Attention"
        )
)
```

---

## 6. Product performance & ABC (Pareto)

```dax
Product Rank =
RANKX ( ALLSELECTED ( DimProduct[Product] ), [Net Revenue], , DESC, DENSE )

Revenue Cumulative % =
VAR CurrentRev = [Net Revenue]
VAR AllProducts =
    ADDCOLUMNS ( ALLSELECTED ( DimProduct[Product] ), "@rev", [Net Revenue] )
VAR TotalRev = SUMX ( AllProducts, [@rev] )
VAR CumRev = SUMX ( FILTER ( AllProducts, [@rev] >= CurrentRev ), [@rev] )
RETURN DIVIDE ( CumRev, TotalRev )

ABC Class =
VAR c = [Revenue Cumulative %]
RETURN
SWITCH ( TRUE (), c <= 0.8, "A", c <= 0.95, "B", "C" )

Top 10 Products Revenue =
CALCULATE ( [Net Revenue], TOPN ( 10, ALLSELECTED ( DimProduct[Product] ), [Net Revenue], DESC ) )

Revenue Concentration % = DIVIDE ( [Top 10 Products Revenue], CALCULATE ( [Net Revenue], ALLSELECTED ( DimProduct[Product] ) ) )

-- like-for-like: only products launched before the comparison period
LFL Revenue =
CALCULATE (
    [Net Revenue],
    FILTER ( VALUES ( DimProduct[LaunchDate] ), DimProduct[LaunchDate] <= MIN ( DimDate[Date] ) - 365 )
)
```

---

## 7. Inventory (semi-additive — the classic trap)

Inventory is a **snapshot**: summing across months double-counts. Use the last
non-blank date in context.

```dax
Units On Hand =
CALCULATE (
    SUM ( FactInventory[UnitsOnHand] ),
    LASTNONBLANK ( DimDate[Date], CALCULATE ( COUNTROWS ( FactInventory ) ) )
)

Inventory Value =
CALCULATE (
    SUM ( FactInventory[InventoryValue] ),
    LASTNONBLANK ( DimDate[Date], CALCULATE ( COUNTROWS ( FactInventory ) ) )
)

Avg Inventory Value = AVERAGEX ( VALUES ( DimDate[YearMonth] ), [Inventory Value] )

Inventory Turnover = DIVIDE ( [Total Cost], [Avg Inventory Value] )

Days of Inventory = DIVIDE ( 365, [Inventory Turnover] )

Sell-Through % =
DIVIDE ( SUM ( FactInventory[UnitsSold] ), SUM ( FactInventory[UnitsSold] ) + [Units On Hand] )

Stock Cover (months) = DIVIDE ( [Units On Hand], AVERAGEX ( VALUES ( DimDate[YearMonth] ), [Units Sold] ) )
```

---

## 8. Marketing efficiency

```dax
Marketing Spend = SUM ( FactMarketingSpend[MarketingSpend] )

ROAS = DIVIDE ( [Net Revenue], [Marketing Spend] )

CAC = DIVIDE ( [Marketing Spend], [New Customers] )

CPC = DIVIDE ( [Marketing Spend], SUM ( FactMarketingSpend[Clicks] ) )

CTR % = DIVIDE ( SUM ( FactMarketingSpend[Clicks] ), SUM ( FactMarketingSpend[Impressions] ) )

Marketing % of Revenue = DIVIDE ( [Marketing Spend], [Net Revenue] )

LTV to CAC = DIVIDE ( [Customer LTV], [CAC] )

Commission Cost =
SUMX ( VALUES ( DimChannel[ChannelKey] ), [Net Revenue] * MAX ( DimChannel[CommissionPct] ) )

Contribution Margin = [Gross Profit] - [Marketing Spend] - [Commission Cost]

Contribution Margin % = DIVIDE ( [Contribution Margin], [Net Revenue] )
```

---

## 9. Senior-level patterns (these are what get noticed)

### Dynamic measure switching
Modeling → **New parameter → Fields**, add Net Revenue, Gross Profit, Orders,
Units Sold, AOV. Drop the parameter on a slicer — one chart now serves five
metrics.

### What-if scenario
Modeling → **New parameter → Numeric range**, name `Discount Scenario %`,
0 to 30, step 1.

```dax
Simulated Revenue =
[Net Revenue] * ( 1 - SELECTEDVALUE ( 'Discount Scenario %'[Discount Scenario %] ) / 100 )

Simulated Gross Profit = [Simulated Revenue] - [Total Cost]

Simulated Margin % = DIVIDE ( [Simulated Gross Profit], [Simulated Revenue] )
```

### Dynamic titles
```dax
Chart Title =
"Revenue by " & SELECTEDVALUE ( DimProduct[Category], "All Categories" )
    & " — " & SELECTEDVALUE ( DimDate[Year], "All Years" )
```
Visual → Title → **fx** → Field value → this measure.

### Conditional formatting by measure
```dax
KPI Colour =
SWITCH (
    TRUE (),
    [Target Attainment %] >= 1,   "#3D6B4A",
    [Target Attainment %] >= 0.9, "#C9BFA8",
    "#8A3B3B"
)
```
Format → Font colour → fx → Field value → `KPI Colour`.

### Handling blanks properly
```dax
Revenue YoY % (safe) =
IF ( ISBLANK ( [Revenue LY] ), BLANK (), DIVIDE ( [Revenue YoY], [Revenue LY] ) )
```
Beginners show +∞ or 100% for the first year. Blanking it is correct.

### Row-level security (mention it, it signals enterprise work)
Modeling → Manage roles → role `Regional Manager` on DimGeography:
```dax
[Region] = USERPRINCIPALNAME ()
```
(or a lookup table). Demonstrate with **View as role**.

---

## 10. Report architecture (6 pages + 2 hidden)

**Page 1 — Executive Overview.** KPI cards (Net Revenue, Gross Profit,
Margin %, Orders, AOV) each with YoY sparkline; revenue trend with MAT line;
attainment gauge; channel mix donut; top-5 products bar.

**Page 2 — Sales Performance.** Revenue trend with YoY overlay; waterfall
(YoY bridge by category); month × category heat matrix; running total.

**Page 3 — Product & ABC.** Pareto chart (bars = revenue, line = cumulative %);
ABC class table with margin; launch-cohort comparison; drill-through enabled.

**Page 4 — Customer Analytics.** Cohort retention matrix; RFM segment
scatter (Frequency × Monetary, size = Recency); new vs returning split;
LTV:CAC by acquisition channel.

**Page 5 — Channel & Marketing ROI.** ROAS by channel; CAC trend;
contribution margin after commission; spend vs revenue scatter.

**Page 6 — Inventory.** Days of inventory by category; sell-through;
stock cover vs velocity quadrant; slow-mover table.

**Hidden — Product Detail (drill-through).** Right-click any product →
drill through. Set drill-through field to `DimProduct[Product]`.

**Hidden — Tooltip page.** Page size = Tooltip; mini trend + KPIs. Assign it
in each visual → Format → Tooltip → Report page.

### Styling (keep the RAW THREAD language)
Canvas `#F7F5F0`, cards `#E8E0D0`, primary data `#1A1A1A`, accent `#C9BFA8`,
positive `#3D6B4A`, negative `#8A3B3B`. Segoe UI. Remove chart borders and
gridlines. Consistent 8px padding.

---

## 11. Publishing (the portfolio payoff)

1. Sign in to Power BI Service (free account works).
2. Home → **Publish** → My workspace.
3. In the Service open the report → **File → Embed report → Publish to web
   (public)** → copy the link.
4. Test it in an incognito window — it must open with **no sign-in**.

That link is the case study's centrepiece: the client interacts with your
report, not a screenshot.

> Publish to web is public — fine for demo data, never for client data.
> If your tenant blocks it, publish the `.pbix` to GitHub and record a GIF
> instead.
