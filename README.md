

# Walmart Beer Category Analysis — Molson Coors 2024

Project Overview

This project simulates the work of a **Category Management Analyst** at Molson Coors Beverage Company, embedded at Walmart's headquarters in Bentonville, AR. Using a synthetic dataset modeled after Circana-style syndicated retail data, this analysis answers five core business questions that a category analyst would bring to a Walmart buyer meeting.

**Tools:** SQL (DB Browser for SQLite) · Excel · Tableau Public  
**Dataset:** 18,200 rows · 14 brands · 5 regions · 25 states · 52 weeks (FY2024)  
**Domain:** CPG / Beverage · Category Management · Retail Analytics

---

## Business Context

Category Management Analysts at major beverage companies like Molson Coors use syndicated data platforms (Circana, NIQ) to track how their brands perform across retail chains like Walmart. Their job is to use data to advise retail buyers on shelf space allocation, promotional strategy, and category growth opportunities — while maximizing their own brand portfolio's share.

This project replicates that workflow end to end.


## Repository Structure
```
Walmart-Beer-Category-Analysis/
│
├── README.md
├── walmart_beer_category.xlsx
└── Walmart Category Analysis.sql
```

**[Download Dataset →](walmart_beer_category.xlsx)**  


## Dataset

The dataset was generated using Python to simulate realistic Walmart beer category POS data with the following structure:

| Column | Description |
|--------|-------------|
| Week | Fiscal week (2024-W01 to 2024-W52) |
| Region | 5 US regions |
| State | 25 states |
| Brand | 14 brands across 6 brand owners |
| Sub_Category | Light Beer, Import, Craft, Hard Seltzer |
| Brand_Owner | Molson Coors, AB InBev, Constellation, etc. |
| Units_Sold | Weekly units sold per store cluster |
| Revenue | Weekly revenue |
| Price_Per_Unit | Effective selling price (includes promo discount) |
| Shelf_Facings | Number of shelf slots assigned to brand |
| On_Promotion | Whether brand was on promotion that week |

**Realistic trends built into the data:**
- Hard Seltzer category decline (-13% H2 vs H1)
- Import category growth (+5.1% H2 vs H1)
- Bud Light volume controversy effect (starting Week 15)
- Promotional price discounts of 10-15% driving ~25% volume lift

---

## SQL Analysis

Five queries were written in DB Browser for SQLite to answer core category management business questions. Full queries available in the SQLite database [`Walmart Category Analysis.db`](Walmart%20Category%20Analysis.db)

---

### Query 1 — Sub-Category Performance Overview
```sql
SELECT
    Sub_Category,
    ROUND(SUM(Revenue) / 1000000.0, 2)        AS Total_Revenue_M,
    SUM(Units_Sold)                             AS Total_Units,
    ROUND(AVG(Price_Per_Unit), 2)              AS Avg_Price,
    ROUND(100.0 * SUM(Revenue) /
        SUM(SUM(Revenue)) OVER (), 1)          AS Revenue_Share_Pct
FROM walmart_beer_category
GROUP BY Sub_Category
ORDER BY Total_Revenue_M DESC;
```

**Finding:** Light Beer dominates at $1.51B (46.5% share), followed by Import at $823M (25.4%). Hard Seltzer ranks last at $426M (13.2%) with the lowest average price per unit ($17.93).

---

### Query 2 — H1 vs H2 Sub-Category Trend
```sql
SELECT
    Sub_Category,
    ROUND(SUM(CASE WHEN Week_Num <= 26
        THEN Revenue ELSE 0 END) / 1000000.0, 2)   AS H1_Revenue_M,
    ROUND(SUM(CASE WHEN Week_Num > 26
        THEN Revenue ELSE 0 END) / 1000000.0, 2)   AS H2_Revenue_M,
    ROUND(100.0 * (
        SUM(CASE WHEN Week_Num > 26  THEN Revenue ELSE 0 END) -
        SUM(CASE WHEN Week_Num <= 26 THEN Revenue ELSE 0 END)
    ) / SUM(CASE WHEN Week_Num <= 26
        THEN Revenue ELSE 0 END), 1)               AS H2_vs_H1_Pct_Change
FROM walmart_beer_category
GROUP BY Sub_Category
ORDER BY H2_vs_H1_Pct_Change DESC;
```

**Finding:** Import (+5.1%) and Craft (+2.9%) grew in H2, while Light Beer (-10.3%) and Hard Seltzer (-13.0%) contracted — signaling a category mix shift toward premium and imported products.

---

### Query 3 — Brand Share & Competitive Benchmarking
```sql
SELECT
    Brand,
    Brand_Owner,
    Sub_Category,
    ROUND(SUM(Revenue) / 1000000.0, 2)        AS Total_Revenue_M,
    SUM(Units_Sold)                             AS Total_Units,
    ROUND(100.0 * SUM(Revenue) /
        SUM(SUM(Revenue)) OVER (), 1)          AS Revenue_Share_Pct,
    ROUND(AVG(Shelf_Facings), 0)               AS Shelf_Facings
FROM walmart_beer_category
GROUP BY Brand, Brand_Owner, Sub_Category
ORDER BY Total_Revenue_M DESC;
```

**Finding:** Coors Light ranked #1 ($344M, 10.6% share) and Miller Lite ranked #3 ($311M, 9.6% share). Bud Light held 7 shelf facings but generated less revenue than Coors Light (6 facings) — a clear shelf efficiency gap. Modelo Especial delivered the highest revenue-per-facing in the entire category.

---

### Query 4 — Promotion Effectiveness Analysis
```sql
SELECT
    Brand,
    Brand_Owner,
    ROUND(AVG(CASE WHEN On_Promotion = 'True'
        THEN Price_Per_Unit END), 2)            AS Promo_Price,
    ROUND(AVG(CASE WHEN On_Promotion = 'False'
        THEN Price_Per_Unit END), 2)            AS Regular_Price,
    ROUND(AVG(CASE WHEN On_Promotion = 'True'
        THEN Units_Sold END), 0)                AS Avg_Units_On_Promo,
    ROUND(AVG(CASE WHEN On_Promotion = 'False'
        THEN Units_Sold END), 0)                AS Avg_Units_Off_Promo,
    ROUND(100.0 * (
        AVG(CASE WHEN On_Promotion = 'True' THEN Units_Sold END) -
        AVG(CASE WHEN On_Promotion = 'False' THEN Units_Sold END)
    ) / AVG(CASE WHEN On_Promotion = 'False'
        THEN Units_Sold END), 1)                AS Volume_Lift_Pct
FROM walmart_beer_category
GROUP BY Brand, Brand_Owner
ORDER BY Volume_Lift_Pct DESC;
```

**Finding:** All 14 brands showed positive promotional lift ranging from 22.4% to 29.4% with an average price reduction of ~$2.10. Miller Lite delivered Molson Coors' strongest response at +28.2% — making it the highest-priority brand for Walmart's weekly ad circular.

---

### Query 5 — Regional Performance & Gap Identification
```sql
SELECT
    Region,
    ROUND(SUM(CASE WHEN Brand_Owner = 'Molson Coors'
        THEN Revenue ELSE 0 END) / 1000000.0, 2)   AS MC_Revenue_M,
    ROUND(SUM(Revenue) / 1000000.0, 2)              AS Total_Category_M,
    ROUND(100.0 *
        SUM(CASE WHEN Brand_Owner = 'Molson Coors'
            THEN Revenue ELSE 0 END) /
        SUM(Revenue), 1)                            AS MC_Share_Pct,
    ROUND(SUM(CASE WHEN Brand_Owner = 'Molson Coors'
        THEN Units_Sold ELSE 0 END), 0)             AS MC_Units,
    ROUND(AVG(CASE WHEN Brand_Owner = 'Molson Coors'
        THEN Price_Per_Unit END), 2)                AS MC_Avg_Price
FROM walmart_beer_category
GROUP BY Region
ORDER BY MC_Share_Pct DESC;
```

**Finding:** Molson Coors maintains consistent 30.2–30.5% revenue share nationally. The West region holds the lowest share (30.2%) in the second-largest beer market ($665M total category revenue) — representing the strongest opportunity for targeted promotional investment.

---

## Tableau Dashboard

**[View Live Dashboard →](https://public.tableau.com/views/WalmartBeerCategoryAnalysisMolsonCoors2024/Dashboard1)**

The dashboard includes 5 visualizations:
- Sub-Category Revenue (bar chart)
- H1 vs H2 Trend (side-by-side bar chart)
- Brand Share (treemap by brand owner)
- Promotion Lift (comparative bar chart)
- Regional Performance (bar chart with average reference line)

---

## Key Recommendations

1. **Reallocate Hard Seltzer facings to Import** — Hard Seltzer is declining 13% H2 vs H1 at the lowest average price in the category. Import brands command $4.74 more per unit and are growing. Shifting 2 facings improves overall category revenue per facing.

2. **Prioritize Miller Lite in Walmart's weekly ad circular** — Miller Lite delivers the strongest promotional volume lift among Molson Coors brands (+28.2%) at a $2.11 average discount. Most efficient promotional investment in the portfolio.

3. **Target West region for incremental investment** — The West is Molson Coors' weakest region (30.2% share) in the second-largest Walmart beer market ($665M). A focused promotional push could recover an estimated $2–3M in incremental annual revenue.

---

## About

Built as a portfolio project simulating Category Management Analyst work at Molson Coors Beverage Company. Dataset is synthetic and generated for analytical demonstration purposes only.

**Author:** Nirbhik Singh  
**LinkedIn:** [linkedin.com/in/nirbhik-singh-734a8221a](https://linkedin.com/in/nirbhik-singh-734a8221a)  
**GitHub:** [github.com/nirbhiksingh7-oss](https://github.com/nirbhiksingh7-oss)
