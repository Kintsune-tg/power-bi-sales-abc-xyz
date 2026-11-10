# Retail Sales Dashboard: ABC-XYZ Product Analysis (Power BI)

A Power BI report built on a synthetic retail dataset. It has two pages: a sales overview (revenue, margin, year-over-year growth) and a product portfolio analysis that combines ABC classification (by revenue) with XYZ classification (by demand stability).

I made this as a portfolio project while moving from ERP / system analysis into data analysis. My main goal was to practice data modeling and DAX, not only to draw charts.

## Data

Four CSV files with synthetic retail data for 2024-2025:

| Table | Content |
|---|---|
| `fact_sales` | order lines: order_id, order_line_id, order_date, customer_id, product_id, channel, quantity, unit_price, discount_pct |
| `dim_product` | 120 products in 8 categories, with unit_price and unit_cost |
| `dim_customer` | customer_id, segment, region, acquisition_channel |
| `dim_calendar` | one row per day (731 days) with year, quarter, month, week and weekday columns |

I cleaned the data in Power Query:

- removed duplicate rows
- set the correct data type for each column
- removed rows with missing values

The raw sales table had 32,106 rows, and 32,006 rows are left after cleaning.

The data is synthetic, so the numbers are not real business results.

## Data model

It is a star schema. `fact_sales` is in the center and three dimensions are connected to it with one-to-many relationships (`dim_calendar` by date, `dim_product` by product_id, `dim_customer` by customer_id).

- `dim_calendar` is marked as a date table. Month names are sorted by month number.
- Technical key columns are hidden from the report view.
- All measures are stored in a separate `_Measures` table.
- `dim_abc` and `dim_xyz` are small helper tables (A/B/C and X/Y/Z) without relationships. I use them only as rows and columns of the ABC-XYZ matrix.
- `fact_sales_raw` (the raw copy of the sales table) is hidden and not used in the report.

## Measures

| Measure | What it does |
|---|---|
| Revenue | quantity × unit_price × (1 − discount_pct), calculated row by row and summed |
| Cost | quantity × unit_cost from `dim_product` |
| Gross Profit, Margin % | revenue minus cost, and its share of revenue |
| Orders, Units | distinct order count and total quantity |
| Avg Order Value | revenue divided by orders |
| Revenue LY, Revenue YoY % | revenue for the same period one year earlier and the change in % |
| Revenue Cum Share % | running share of total revenue, products sorted from the biggest |
| ABC Class | A: up to 80% of cumulative revenue, B: up to 95%, C: the rest |
| Units Avg per Month, Units StdDev per Month | mean and standard deviation of monthly units per product |
| Demand CV | standard deviation divided by mean (coefficient of variation) |
| XYZ Class | X: CV below 0.5, Y: below 1.0, Z: 1.0 and above |
| Products in Cell | number of products in each ABC × XYZ cell of the matrix |

A few of them in DAX:

```dax
Revenue =
SUMX ( fact_sales, fact_sales[quantity] * fact_sales[unit_price] * ( 1 - fact_sales[discount_pct] ) )
```

```dax
Revenue Cum Share % =
IF (
    HASONEVALUE ( dim_product[product_name] ),
    VAR CurrentRevenue = [Revenue]
    VAR AllProducts = ALLSELECTED ( dim_product )
    VAR CumRevenue =
        SUMX ( FILTER ( AllProducts, [Revenue] >= CurrentRevenue ), [Revenue] )
    VAR TotalRevenue = CALCULATE ( [Revenue], AllProducts )
    RETURN
        DIVIDE ( CumRevenue, TotalRevenue )
)
```

```dax
Units Avg per Month =
AVERAGEX ( VALUES ( dim_calendar[year_month] ), COALESCE ( [Units], 0 ) )

Units StdDev per Month =
STDEVX.P ( VALUES ( dim_calendar[year_month] ), COALESCE ( [Units], 0 ) )

Demand CV =
DIVIDE ( [Units StdDev per Month], [Units Avg per Month] )
```

A month without sales counts as zero (`COALESCE`). Without it, a product sold in only a few months would look stable.

## What the report shows

**Overview page.** KPI cards (revenue 7.82M, about 17K orders, margin 36.8%), revenue against the previous year, year-over-year change by month (green, yellow and red bars), revenue by category and top 10 products. Slicers: year, segment, region, category.

**ABC-XYZ page.** ABC × XYZ matrix with product counts, Pareto chart for the products that make up the first 80% of revenue, a bubble chart (demand variability vs revenue) and a detailed product table.

## Findings

- 16 of 120 products (13%) generate 80% of revenue (class A). 70 products (58%) bring only about 5% (class C).
- 11 of 16 A-class products are in Electronics (4 are Furniture and 1 is Home Appliances). The revenue depends heavily on one category.
- Only 4 products are in AX (high revenue, stable demand). 34 products (28% of the assortment) are in CZ: low revenue and unpredictable demand. They are the first candidates for review.
- Revenue in 2025 was about 15% higher than in 2024. Two months were below the previous year: July and November.
- Demand CV for all products together is 0.24, while single products have much higher values. The standard deviation of a total is not the sum of the standard deviations: ups and downs of different products partly cancel each other.

## Things I learned

- A filter on a measure at the visual level is applied before `ALLSELECTED`, so my cumulative share on the Pareto chart reached 100% after I filtered it to 80%. I fixed it by returning blank inside the measures (`Revenue (to 80%)` and `Cum Share (to 80%)`) instead of filtering the visual.
- The XYZ thresholds are 0.5 and 1.0, not the classic 10% and 25%. Monthly demand per product in this dataset is noisy, and with classic thresholds almost every product would be Z.
- Measures are calculated again in every cell, including the total row. This is why the total standard deviation is not a sum.

## How I built it

I am learning DAX, and I used an AI assistant (Claude) as a tutor while building this report. It explained functions and filter context, and I compared every measure with its SQL equivalent. I wrote the first versions of some measures myself (for example Demand CV and XYZ Class) and fixed them after review. I checked the results with control totals: row counts after cleaning, the sum of the matrix cells equals 120 products, and the average units per month times 24 months matches total units.

## How to open it

1. Open `Power BI Sales Report.pbix` in Power BI Desktop.
2. The report reads the CSV files from a local folder. If Power BI cannot find them, go to Home → Transform data → Data source settings and change the path to the `data` folder of this repository. Then click Refresh.

## Repository structure

```
.
├── Power BI Sales Report.pbix
├── data/                  CSV files
├── screenshots/           report pages
└── README.md
```

## Next steps

I plan to repeat the XYZ calculation in Python (pandas) and compare it with the DAX results.

## Contact

Ivan Aristov, Tbilisi, Georgia
LinkedIn: https://www.linkedin.com/in/aristovivan
GitHub: https://github.com/Kintsune-tg
Telegram: @Kintsune
Phone Number: +995 595 056 458
What's App: +7 982 649 31 36
