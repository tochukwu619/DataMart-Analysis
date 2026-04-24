# Data Mart: Sustainable Packaging Impact Analysis
### Case Study | SQL Data Analysis | June 2020

---

## Background

Data Mart is an online supermarket specialising in fresh produce. In **June 2020**, Data Mart made a large-scale operational shift by switching to **sustainable packaging** across every step of their supply chain. The key date of this change was **2020-06-15**.

This report investigates the quantifiable impact of that change, identifies which business segments were most affected, and recommends strategies for managing future sustainability transitions with minimal disruption to sales.

---

## Dataset Overview

The analysis is based on the `weekly_sales` table, which was cleaned and transformed into `clean_weekly_sales` with the following schema:

| Column | Type | Description |
|---|---|---|
| `week_date` | DATE | Weekly sales period start date |
| `region` | VARCHAR(13) | Sales region |
| `platform` | VARCHAR(7) | Retail or Shopify |
| `segment` | VARCHAR(4) | Customer segment code |
| `customer_type` | VARCHAR(8) | New, Existing, or Guest |
| `transactions` | INTEGER | Number of transactions |
| `sales` | INTEGER | Total sales value |
| `week_number` | INTEGER | Week of year |
| `month_number` | INTEGER | Month of year |
| `calendar_year` | INTEGER | Year |
| `age_band` | VARCHAR(50) | Young Adults, Middle Aged, Retirees |
| `demographics` | VARCHAR(50) | Couples, Families |
| `avg_transaction` | FLOAT | Sales divided by transactions |

---

## Tools & Methodology

- **Database:** Microsoft SQL Server (T-SQL)
- **Techniques Used:** CTEs, Window Functions, CASE expressions, DATEADD/DATEPART, CAST for type safety, recursive CTEs for gap analysis, view creation for reusable analysis
- **Analysis Framework:** Exploratory Data Analysis → Before & After Comparison → Year-on-Year Benchmarking

---


## Data Transformation

Before analysis, the raw `weekly_sales` table underwent several cleaning steps:

- `week_date` was converted from string to a proper `DATE` type using `TRY_CONVERT(DATE, week_date, 3)`
- Derived columns were added: `week_number`, `month_number`, `calendar_year`
- `age_band` was mapped from the last character of `segment`:
  - `'1'` → Young Adults
  - `'2'` → Middle Aged
  - All others → Retirees
  - `'null'` → NULL
- `demographics` was mapped from the first character of `segment`:
  - `'C'` → Couples
  - `'F'` → Families
  - All others → NULL
- `avg_transaction` was calculated as `ROUND(sales / transactions, 2)`
- Segment values of `'null'` (string) were replaced with actual `NULL`

---

## Exploratory Data Analysis

### Q1 — What day of the week is used for each `week_date` value?

```sql
SELECT DATENAME(WEEKDAY, week_date) AS Day_of_the_week
FROM clean_weekly_sales;
```

**Result:** All `week_date` values fall on a **Monday**. The dataset uses Monday as the consistent weekly anchor point.

---

### Q2 — What range of week numbers are missing from the dataset?

```sql
WITH NumberRange AS (
    SELECT 1 AS MyNumber
    UNION ALL
    SELECT MyNumber + 1 FROM NumberRange WHERE MyNumber < 52
)
SELECT MyNumber FROM NumberRange
WHERE MyNumber NOT IN (SELECT week_number FROM clean_weekly_sales);
```

**Result:** Weeks **1–12** and **37–52** are absent, meaning the dataset covers **weeks 13 through 36** — roughly March to September across the recorded years.

---

### Q3 — How many total transactions were there for each year?

```sql
SELECT calendar_year, SUM(transactions) AS total_transactions
FROM clean_weekly_sales
GROUP BY calendar_year
ORDER BY 1;
```

| Year | Total Transactions |
|------|--------------------|
| 2018 | 346,406,460 |
| 2019 | 365,639,285 |
| 2020 | 375,813,651 |

Transaction volumes grew year-over-year, with **2020 recording the highest** volume despite the packaging change mid-year.

---

### Q4 — What is the total sales for each region for each month?

Sales were broken down by region and month using:

```sql
SELECT region, month_number, DATENAME(MONTH, week_date) AS Month_name,
       SUM(CAST(sales AS BIGINT)) AS Total_sales
FROM clean_weekly_sales
GROUP BY region, month_number, DATENAME(MONTH, week_date)
ORDER BY region, month_number;
```

This gives a full monthly sales matrix per region, which is useful for spotting regional seasonality and the post-June impact by geography.

---

### Q5 — What is the total count of transactions for each platform?

```sql
SELECT platform, SUM(transactions) AS Total_transactions
FROM clean_weekly_sales
GROUP BY platform;
```

| Platform | Total Transactions |
|----------|--------------------|
| Retail | 1,081,934,227 |
| Shopify | 5,925,169 |

**Retail dominates** with over 99% of all transactions. Shopify is a smaller but growing channel.

---

### Q6 — What is the percentage of sales for Retail vs Shopify each month?

| Month | Retail % | Shopify % |
|-------|----------|-----------|
| March | 97.54% | 2.46% |
| April | 97.59% | 2.41% |
| May | 97.30% | 2.70% |
| June | 97.27% | 2.73% |
| July | 97.29% | 2.71% |
| August | 97.08% | 2.92% |
| September | 97.38% | 2.62% |

Retail consistently accounts for **~97%** of monthly sales. Shopify's share shows a slight uptick in August, possibly reflecting a shift post-packaging change.

---

### Q7 — What is the percentage of sales by demographic for each year?

| Year | Couples % | Families % | Unknown % |
|------|-----------|------------|-----------|
| 2018 | 26.38% | 31.99% | 41.63% |
| 2019 | 27.28% | 32.47% | 40.25% |
| 2020 | 28.72% | 32.73% | 38.55% |

The **unknown demographic** segment is declining year-on-year, while Couples and Families are growing. This suggests improving data capture over time.

---

### Q8 — Which age band and demographic contribute most to Retail sales?

| Age Band | Couples % | Families % | Unknown % |
|----------|-----------|------------|-----------|
| Unknown | 0.00% | 0.00% | 100.00% |
| Young Adults | 59.51% | 40.49% | 0.00% |
| Middle Aged | 29.87% | 70.13% | 0.00% |
| Retirees | 48.98% | 51.02% | 0.00% |

**Middle Aged Families** are the dominant known contributor to Retail sales, followed by **Retirees** (split near-evenly between Couples and Families).

---

### Q9 — Can `avg_transaction` be used to find average transaction size per year for Retail vs Shopify?

The `avg_transaction` column represents a **per-row average** (sales ÷ transactions per weekly record), not a true aggregate average. To get the correct yearly average, sales and transactions must be summed first:

```sql
SELECT calendar_year,
       ROUND(SUM(CASE WHEN platform = 'Retail' THEN sales ELSE 0 END) * 1.0 /
             SUM(CASE WHEN platform = 'Retail' THEN transactions ELSE 0 END), 2) AS retail_avg_transaction,
       ROUND(SUM(CASE WHEN platform = 'Shopify' THEN sales ELSE 0 END) * 1.0 /
             SUM(CASE WHEN platform = 'Shopify' THEN transactions ELSE 0 END), 2) AS shopify_avg_transaction
FROM clean_weekly_sales
GROUP BY calendar_year ORDER BY calendar_year;
```

| Year | Retail Avg ($) | Shopify Avg ($) |
|------|----------------|-----------------|
| 2018 | 36.56 | 192.49 |
| 2019 | 36.83 | 183.36 |
| 2020 | 36.56 | 179.03 |

**Shopify customers spend ~5× more per transaction** than Retail customers, though Shopify average transaction values are declining slightly. The `avg_transaction` column alone should **not** be used for this aggregation due to the averaging-of-averages problem.

---

## Before & After Analysis

**Context:** The sustainable packaging change was introduced on **2020-06-15**. The following analysis compares sales performance before and after this date to measure the impact.

---

### B&A Q1 — 4 Weeks Before vs After 2020-06-15

```sql
WITH period_sales AS (
    SELECT
        CASE
            WHEN week_date BETWEEN DATEADD(WEEK, -4, '2020-06-15') AND DATEADD(DAY, -1, '2020-06-15') THEN 'Before'
            WHEN week_date BETWEEN '2020-06-15' AND DATEADD(WEEK, 3, '2020-06-15') THEN 'After'
        END AS period,
        SUM(CAST(sales AS BIGINT)) AS total_sales
    FROM clean_weekly_sales
    WHERE week_date BETWEEN DATEADD(WEEK, -4, '2020-06-15') AND DATEADD(WEEK, 3, '2020-06-15')
    GROUP BY CASE
        WHEN week_date BETWEEN DATEADD(WEEK, -4, '2020-06-15') AND DATEADD(DAY, -1, '2020-06-15') THEN 'Before'
        WHEN week_date BETWEEN '2020-06-15' AND DATEADD(WEEK, 3, '2020-06-15') THEN 'After'
    END
),
calculation AS (
    SELECT
        MAX(CASE WHEN period = 'Before' THEN total_sales END) AS sales_before,
        MAX(CASE WHEN period = 'After' THEN total_sales END) AS sales_after
    FROM period_sales
)
SELECT sales_before, sales_after,
       (sales_after - sales_before) AS actual_diff,
       ROUND(100.0 * (sales_after - sales_before) / sales_before, 2) AS percentage_change
FROM calculation;
```

| Sales Before | Sales After | Actual Diff | % Change |
|--------------|-------------|-------------|----------|
| 2,345,878,357 | 2,318,994,169 | -26,884,188 | **-1.15%** |

In the 4 weeks immediately following the change, sales dropped by approximately **$26.9 million (-1.15%)**.

---

### B&A Q2 — 12 Weeks Before vs After 2020-06-15

Expanding the window to 12 weeks on each side:

| Sales Before | Sales After | Actual Diff | % Change |
|--------------|-------------|-------------|----------|
| 7,126,273,147 | 6,973,947,753 | -152,325,394 | **-2.14%** |

Over a 12-week window, the decline deepens to **$152.3 million (-2.14%)**, suggesting the negative impact compounded over time rather than recovering quickly.

---

### B&A Q3 — Comparing 2020 to Baseline Years (2018 & 2019)

To determine whether the 2020 decline is attributable to the packaging change or is part of a broader trend, the same week-number windows were applied across all three years.

```sql
-- Views created: Analysis4Weeks and Analysis12Weeks
-- Comparing equivalent week ranges in 2018, 2019, and 2020
SELECT * FROM Analysis4Weeks
UNION ALL
SELECT * FROM Analysis12Weeks;
```

| Metric | Year | Sales Before | Sales After | Actual Diff | % Change |
|--------|------|--------------|-------------|-------------|----------|
| 4-Weeks | 2018 | 2,125,140,809 | 2,129,242,914 | +4,102,105 | **+0.19%** |
| 4-Weeks | 2019 | 2,249,989,796 | 2,252,326,390 | +2,336,594 | **+0.10%** |
| 4-Weeks | 2020 | 2,345,878,357 | 2,318,994,169 | -26,884,188 | **-1.15%** |
| 12-Weeks | 2018 | 6,396,562,317 | 6,500,818,510 | +104,256,193 | **+1.63%** |
| 12-Weeks | 2019 | 6,883,386,397 | 6,862,646,103 | -20,740,294 | **-0.30%** |
| 12-Weeks | 2020 | 7,126,273,147 | 6,973,947,753 | -152,325,394 | **-2.14%** |

**Key finding:** In both 2018 and 2019, the equivalent period showed **positive or near-flat growth**. The sharp **-2.14% decline in 2020** is a clear anomaly attributable to the packaging change.

---

## Key Findings & Answers to Business Questions

### 1. What was the quantifiable impact of the June 2020 packaging change?

The packaging change had a **measurable negative impact on sales**:

- In the **4 weeks** following the change: **-$26.9M (-1.15%)**
- In the **12 weeks** following the change: **-$152.3M (-2.14%)**

Compared to baseline years (2018: +1.63%, 2019: -0.30%), the 2020 12-week decline of **-2.14% is a significant outlier**, strongly attributable to the packaging transition.

---

### 2. Which platform, region, segment, and customer types were most impacted?

Based on the EDA findings:

- **Platform:** The **Retail** channel, accounting for ~97.5% of all sales, would absorb the majority of the absolute impact. Shopify, while higher in average transaction value, represents too small a share to significantly buffer overall losses.
- **Demographics:** The **unknown demographic** (38.55% of 2020 sales) represents a significant gap in customer visibility. The largest known groups, **Middle Aged Families** (70.13% of Retail sales) and **Retirees**, are most likely to be price-sensitive and habitual buyers who may react negatively to packaging changes.
- **Age Band:** **Middle Aged** customers show the strongest family demographic alignment and are likely the most impacted, given their dominance in Retail.
- **Transaction Size:** Shopify customers (~5× higher average transaction) may be more environmentally conscious and potentially *less* impacted or even positively influenced by the sustainability move.

---

### 3. What can be done to minimise the impact of future sustainability updates?

Based on this analysis, the following recommendations are proposed:

** Phased Rollout**
Instead of a single cutover date, introduce packaging changes gradually by region or platform. This limits simultaneous exposure across all customer segments and allows early feedback before full deployment.

** Advance Customer Communication**
Proactively communicate the sustainability change to customers, particularly the Retail customer base (Families, Middle Aged, Retirees), through targeted campaigns before the change date. Framing the update as a values-aligned decision can reduce negative reactions.

** Segment-Specific Monitoring**
Set up pre/post dashboards segmented by demographics, platform, and region. This allows early detection of which groups are most affected and enables swift corrective action (e.g., loyalty incentives for high-impact segments).

** Shopify Leverage**
Shopify customers tend to have higher spend per transaction and may be more receptive to sustainable messaging. Leading the campaign with the Shopify channel could generate positive brand momentum before the full Retail rollout.

** Establish A/B Baselines**
Run controlled regional pilots before company-wide changes. This creates a cleaner comparison group and enables more precise attribution of any sales impact.


