# Project 5: AISC Dashboard - DAX Measures Documentation

This document explains all DAX measures created for the AISC Cost-Per-Tonne Dashboard, including business logic and formulas.

---

## 1. AISC per Tonne

**Purpose:** Calculate the All-In Sustaining Cost per tonne of ore produced. This is the PRIMARY metric for mining cost competitiveness.

**Business Logic:** 
- Total costs (all categories) divided by total tonnes produced
- Handles the data structure where tonnes are duplicated 4x (once per cost category)
- Uses UniqueTonnes calculated column to avoid 4x counting

**Formula:**
```dax
AISC per Tonne = 
DIVIDE(
    SUM(FactCosts[TotalCost_Actual]),
    SUM(FactCosts[UniqueTonnes])
)
```

**Why This Matters:**
- AISC is the gold-standard metric for mining company health
- Bottom-quartile AISC producers survive commodity downturns
- Used by investors, CFOs, and mine managers for strategic decisions

**Format:** Currency ($ AUD), 2 decimal places

---

## 2. Total Tonnes

**Purpose:** Sum of total tonnes produced across selected data.

**Business Logic:** Simple aggregation of the TonnesProduced column.

**Formula:**
```dax
Total Tonnes = SUM(FactCosts[TonnesProduced])
```

**Why This Matters:**
- Production volume is a key operational KPI
- Used in AISC calculation and productivity analysis
- Higher production spreads fixed costs over more units

**Format:** Whole number with thousands separator

---

## 3. Budget Variance

**Purpose:** Calculate the dollar difference between actual costs and budgeted costs.

**Business Logic:** 
- Actual costs minus budget costs
- Positive = over budget (bad)
- Negative = under budget (good)

**Formula:**
```dax
Budget Variance = 
SUM(FactCosts[TotalCost_Actual]) - SUM(FactCosts[TotalCost_Budget])
```

**Why This Matters:**
- Core financial control metric
- Identifies cost overruns requiring management intervention
- Used in monthly/quarterly financial reporting

**Format:** Currency ($ AUD), 0 decimal places

---

## 4. Budget Variance %

**Purpose:** Express budget variance as a percentage of the budget for easier comparison.

**Business Logic:** 
- Variance divided by budget
- Uses DIVIDE() to prevent divide-by-zero errors

**Formula:**
```dax
Budget Variance % = 
DIVIDE(
    [Budget Variance],
    SUM(FactCosts[TotalCost_Budget])
)
```

**Why This Matters:**
- Percentages allow fair comparison across sites with different cost bases
- Industry standard: ±5% variance is acceptable, >10% requires action
- Triggers for conditional formatting (green/red alerts)

**Format:** Percentage, 2 decimal places

---

## 5. AISC YoY %

**Purpose:** Calculate year-over-year change in AISC to track cost improvement or deterioration.

**Business Logic:**
- Compares current year AISC to previous year AISC
- Returns percentage change
- Negative % = improvement (lower cost)
- Positive % = deterioration (higher cost)

**Formula:**
```dax
AISC YoY % = 
VAR CurrentYear = MAX(DimDate[Year])
VAR CurrentYearAISC = [AISC per Tonne]
VAR PreviousYearAISC = 
    CALCULATE(
        [AISC per Tonne],
        DimDate[Year] = CurrentYear - 1
    )
RETURN
DIVIDE(
    CurrentYearAISC - PreviousYearAISC,
    PreviousYearAISC
)
```

**Why This Matters:**
- Tracks continuous improvement initiatives
- Board-level KPI for operational efficiency
- Mining companies target 2-5% annual AISC reduction

**Format:** Percentage, 2 decimal places

---

## 6. Cost Ranking

**Purpose:** Rank mine sites from best (1) to worst (3+) based on AISC performance.

**Business Logic:**
- Uses RANKX to rank all sites
- ASC order = lowest AISC gets Rank 1 (best)
- Updates dynamically based on filters

**Formula:**
```dax
Cost Ranking = 
RANKX(
    ALL(DimSite[SiteName]),
    [AISC per Tonne],
    ,
    ASC
)
```

**Why This Matters:**
- Identifies best-practice sites for benchmarking
- Determines which sites are competitive vs. at-risk
- Bottom-quartile ranking = competitive advantage during commodity downturns

**Format:** Whole number

---

## 7. Avg Ore Grade

**Purpose:** Calculate average ore grade percentage across selected data.

**Business Logic:**
- Simple average of OreGrade column
- Divided by 100 for correct percentage display

**Formula:**
```dax
Avg Ore Grade = AVERAGE(FactCosts[OreGrade]) / 100
```

**Why This Matters:**
- Ore grade directly impacts processing costs
- Lower grade = higher $/tonne processing costs
- Critical variable for mine planning and sensitivity analysis

**Format:** Percentage, 2 decimal places

---

## 8. Cost per Category

**Purpose:** Calculate cost per tonne broken down by cost category (Mining, Processing, G&A, CapEx).

**Business Logic:**
- Calculates cost for the specific category in context
- Divides by total tonnes (avoiding 4x counting)
- Used in waterfall and breakdown charts

**Formula:**
```dax
Cost per Category = 
VAR CategoryCost = SUM(FactCosts[TotalCost_Actual])
VAR TotalTonnes = 
    CALCULATE(
        SUM(FactCosts[UniqueTonnes]),
        ALLEXCEPT(FactCosts, FactCosts[SiteKey], FactCosts[DateKey])
    )
RETURN
DIVIDE(CategoryCost, TotalTonnes)
```

**Why This Matters:**
- Shows AISC composition (which categories drive costs)
- Identifies cost reduction opportunities
- Processing + Mining = ~84% of AISC (focus areas for optimization)

**Format:** Currency ($ AUD), 2 decimal places

---

## 9. AISC Impact

**Purpose:** Show the dollar impact of ore grade changes on AISC (used in sensitivity analysis).

**Business Logic:**
- Difference between adjusted AISC and base AISC
- Negative = cost reduction
- Positive = cost increase

**Formula:**
```dax
AISC Impact = [AISC with Sensitivity] - [AISC per Tonne]
```

**Why This Matters:**
- Quantifies value of ore grade improvements
- Supports capital investment decisions (e.g., improved ore sorting)
- Shows vulnerability to ore grade decline

**Format:** Currency ($ AUD), 2 decimal places

---

## 10. AISC with Sensitivity

**Purpose:** Calculate adjusted AISC based on user-selected ore grade adjustment from slider.

**Business Logic:**
- Reads slider value (-0.5 to +0.5)
- Applies 12% cost impact per 1% ore grade change
- Higher ore grade = lower processing cost = lower AISC

**Formula:**
```dax
AISC with Sensitivity = 
VAR Adjustment = SELECTEDVALUE('Ore Grade Adjustment'[Ore Grade Adjustment], 0)
RETURN [AISC per Tonne] * (1 + (Adjustment * -0.12))
```

**Why This Matters:**
- Interactive what-if modeling for strategic planning
- Shows business case for ore grade improvement initiatives
- Models downside risk if ore grade declines

**Format:** Currency ($ AUD), 2 decimal places

---

## Calculated Column: UniqueTonnes

**Purpose:** Prevent 4x counting of tonnes in data structure.

**Business Logic:**
- Data has 4 rows per month/site (one per cost category)
- All 4 rows have same TonnesProduced value
- This column captures tonnes only once (when CostCategoryKey = 0)

**Formula:**
```dax
UniqueTonnes = 
IF(
    FactCosts[CostCategoryKey] = 0,
    FactCosts[TonnesProduced],
    0
)
```

**Why This Matters:**
- Essential for correct AISC calculation
- Without this, AISC would be divided by 4x actual tonnes
- Data modeling solution to handle star schema structure

**Format:** Decimal number, 2 decimal places

---

## DAX Best Practices Applied:

1. **DIVIDE() instead of division operator** - Prevents divide-by-zero errors
2. **Variables (VAR)** - Improves readability and performance
3. **SELECTEDVALUE()** - Safely handles slicer selections
4. **CALCULATE() for context transitions** - Filters data correctly in measures
5. **ALL() and ALLEXCEPT()** - Controls filter context for rankings and comparisons

---

## Key Relationships Used:

- **FactCosts → DimSite** via SiteKey
- **FactCosts → DimDate** via DateKey  
- **FactCosts → DimCostCategory** via CostCategoryKey
- **One-to-many relationships** (star schema design)

---

## Business Context:

### AISC Industry Benchmarks:
- **Bottom quartile (competitive):** <$190/tonne
- **Second quartile (acceptable):** $190-200/tonne
- **Third quartile (at risk):** $200-210/tonne
- **Top quartile (high cost):** >$210/tonne

### Key Cost Drivers:
1. **Processing costs:** 46-48% of total AISC (largest component)
2. **Mining costs:** 36-37% of total AISC
3. **G&A:** 9-10% of total AISC
4. **Sustaining CapEx:** 6-8% of total AISC

### Operational Insights:
- **Ore grade sensitivity:** 1% grade improvement ≈ $12/tonne AISC reduction
- **Seasonal impact:** Q1 costs average +8% vs. Q2-Q4 (cyclone season in Pilbara)
- **Budget tolerance:** ±5% variance acceptable, >10% requires management action

---

**End of DAX Documentation**

---

## Project Information

**Author:** Flavio  
**Project:** AISC Cost-Per-Tonne Dashboard  
**Tech Stack:** Power BI, DAX, SQLite, Python  
**Target Industry:** Mining & Resources (Perth, WA)  
**Date:** February 2026
