# RAW THREAD — Data Warehouse (Power BI source)

Synthetic star schema for a fictional streetwear brand. 3 years of daily
transactions (2023-01-01 → 2025-12-31). **Demo data — no real customers.**

Total: **$18.3M revenue · 78,249 sales lines · 46,000 orders · 3,981 customers**
YoY growth: 2024 **+32.8%**, 2025 **+22.9%** · Gross margin **53.5%**

## Schema

```
                    DimDate (1,096)
                        |
   DimProduct (30) — FactSales (78,249) — DimCustomer (4,200)
                     /      |      \
        DimChannel (4)  DimGeography (15)
                        |
   FactTargets (864) · FactInventory (977) · FactMarketingSpend (144)
```

All relationships are **one-to-many, single direction** (dimension → fact).
Facts sit at different grains on purpose — sales are daily, targets /
inventory / marketing are month-end — so the model needs proper handling
rather than naive joins.

## Dimensions

**DimDate** — 1,096 rows. Full calendar: Year, Quarter, MonthNo, MonthName,
YearMonth, Day, DayName, DayOfWeek, IsWeekend, WeekOfYear, DayOfYear,
MonthEnd, plus **FiscalYear / FiscalQuarter** (fiscal year starts 1 Feb) and
IsHolidaySeason. Mark as date table on `Date`.

**DimProduct** — 30 rows. Hierarchy **Category → Subcategory → Product**,
SKU, ListPrice, StandardCost, GrossMarginPct, Collection (Core / SS Drop /
FW Drop / Tech) and **LaunchDate** — products enter the range at different
times, which makes like-for-like comparisons non-trivial.

**DimCustomer** — 4,200 rows. Segment (Retail / VIP / Wholesale Account),
**AcquisitionDate** + AcquisitionYearMonth (cohort key), AcquisitionChannel,
GeographyKey.

**DimChannel** — 4 rows. Online Store, Wholesale, Pop-up, Marketplace, with
ChannelGroup (Direct / B2B / Partner) and CommissionPct.

**DimGeography** — 15 rows. Hierarchy **Region → Country → City**.

## Facts

**FactSales** — 78,249 lines. Order-level detail: Quantity, UnitPrice,
DiscountPct, Revenue, Cost, plus **ReturnedQty / ReturnedAmount**
(≈4.2% of retail lines) so net-of-returns measures are possible.

**FactTargets** — 864 rows. Monthly TargetRevenue and TargetMargin by
**Category × Channel** — budget-vs-actual at a coarser grain than sales.

**FactInventory** — 977 rows. Month-end snapshot per product:
UnitsOnHand, UnitsReceived, UnitsSold, InventoryValue. Semi-additive —
must not be summed across dates.

**FactMarketingSpend** — 144 rows. Monthly spend by channel, with
Impressions and Clicks — enables ROAS, CAC, CPC.

## Patterns built into the data

- **Seasonality** — Nov/Dec peak (×1.5), slow Jan/Feb; Hoodies & Jackets
  rise in cold months, T-Shirts & Caps in warm ones
- **Growth** — ~+25%/yr compounding
- **Channel economics** — Wholesale: few orders, large baskets, ~45% lower
  unit price; Marketplace carries 15% commission
- **Customer lifecycle** — acquisition dates spread across all 3 years,
  enabling cohort retention analysis
- **Product lifecycle** — staggered LaunchDates (Tech collection only from
  Mar 2024) — new vs mature product comparisons
- **Discounting** — heavier in Nov/Dec, so margin ≠ revenue trend
