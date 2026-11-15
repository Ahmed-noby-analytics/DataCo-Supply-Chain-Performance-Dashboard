# DataCo Supply Chain Performance Dashboard

An interactive Power BI dashboard built with Power Query and DAX to analyze sales, delivery performance, and profitability for DataCo Global, using the DataCo Smart Supply Chain dataset.

The project explores revenue, profit, late deliveries, shipping modes, product categories, and loss-making orders. The goal is to turn raw order data into insights that can support decisions on shipping, delivery promises, and pricing.

![Welcome Page](images/welcome_page.png)

## Table of Contents

1. [Business Problem](#1-business-problem)
2. [Business Questions](#2-business-questions)
3. [Dataset and Source](#3-dataset-and-source)
4. [Data Cleaning and Validation](#4-data-cleaning-and-validation)
5. [Data Model](#5-data-model)
6. [Metric Definitions](#6-metric-definitions)
7. [Dashboard Overview](#7-dashboard-overview)
8. [Key Results](#8-key-results)
9. [Key Insights](#9-key-insights)
10. [Business Recommendations](#10-business-recommendations)
11. [Limitations](#11-limitations)
12. [Tools Used](#12-tools-used)
13. [Repository Contents](#13-repository-contents)

---

## 1. Business Problem

DataCo Global sells sports, fitness, and consumer products to customers across many markets. As order volume grows, the business needs a clear view of how sales, deliveries, and profitability are performing.

This analysis focuses on understanding where sales come from, how often orders arrive late and what drives those delays, and where the business loses money on orders.

The objective is to give a consolidated overview of supply chain performance and to highlight opportunities for improving delivery reliability and profitability.

## 2. Business Questions

The project addresses the following business questions:

1. What are total sales, profit, and profit margin, and how do they change over the months?
2. Which markets, customer segments, and product categories generate the most sales?
3. How often do orders arrive late?
4. Is late delivery a regional problem or a shipping-mode problem?
5. How does the actual shipping time compare with the scheduled shipping time?
6. Which categories and products earn the most profit, and which lose money?
7. How many orders lose money, and does discounting explain the losses?

## 3. Dataset and Source

The dataset is the **DataCo Smart Supply Chain for Big Data Analysis** dataset, available on Kaggle and Mendeley Data. It contains order-level records of sales, shipping, customers, and products.

The files in the download:

| File | Description | Used in project |
|---|---|---|
| `DataCoSupplyChainDataset.csv` | Main dataset, one row per order item (about 180K rows, 50+ columns) | Yes |
| `DescriptionDataCoSupplyChain.csv` | Description of the columns | Reference only |
| `tokenized_access_logs.csv` | Website clickstream logs | No |

**Period used in the dashboard:** 2015 – 2017. The dataset also contains a small number of orders from January and the beginning of February 2018, which were excluded because the year is incomplete.

Please check the license on the dataset's source page before reusing it. This project is for portfolio and learning purposes.

## 4. Data Cleaning and Validation

Before building the dashboard, I reviewed the data in Power Query using column quality and column profile.

| Check | Result |
|---|---|
| File encoding | Loaded with Western European (1252) encoding to keep Spanish characters correct |
| Table naming | Main table renamed to `Orders` |
| Grain | Each row is an order item, not an order. `Order Id` repeats, so orders are counted with a distinct count |
| Personal information | Customer name, email, password, and street columns were removed (they are masked in the source and not needed for analysis) |
| Unused columns | Product description, product image, and the duplicate `Benefit per order` column were removed |
| Order Zipcode | Removed because it is mostly empty |
| Shipping date | Removed because it mixes two different date formats. Delivery analysis relies on `Days for shipping (real)` and `Days for shipment (scheduled)` instead |
| Order date | Converted to Date using the English (United States) locale, with the time removed so it can relate to the calendar table |
| Country names | `EE. UU.` replaced with `USA` in `Customer Country` |
| Data types | Reviewed and corrected where needed (dates as Date, sales and profit as Decimal, quantity and day columns as Whole Number) |
| Time range | Rows from 2018 filtered out because the year is incomplete, so the Year slicer shows 2015 – 2017 only |

The applied steps in Power Query: Promoted Headers, Changed Type, Removed Columns, Renamed Columns, Changed Type with Locale, Replaced Value, and Filtered Rows.

These checks established a consistent foundation for the data model and the analysis.

![Power Query Applied Steps](images/power_query_steps.png)

## 5. Data Model

The data model is intentionally simple: one fact table, one date table, and one table that holds the measures.

| Table | Role |
|---|---|
| `Orders` | Single flat table with order items, products, customers, shipping, and profit columns |
| `Calendar` | Date table created with DAX, marked as the date table, with Year, Quarter, Month, and Month Number |
| `Measure` | Dedicated table that stores all DAX measures (no relationships) |

| From (one side) | To (many side) | Key |
|---|---|---|
| `Calendar` | `Orders` | `Date` → `Order Date` |

The relationship is one-to-many with a single filter direction. The calendar is built from the first to the last year of order dates, and the month column is sorted by month number.

![Data Model](images/data_model.png)

## 6. Metric Definitions

| Metric | Definition |
|---|---|
| Total Sales | Sum of `Sales` |
| Total Profit | Sum of `Order Profit Per Order` |
| Profit Margin % | Total Profit ÷ Total Sales |
| Orders | Distinct count of `Order Id` |
| Late Delivery % | Average of `Late_delivery_risk` (a 0/1 flag in the dataset, 1 = late), calculated on order lines |
| Avg Real Days | Average of `Days for shipping (real)` |
| Avg Scheduled Days | Average of `Days for shipment (scheduled)` |
| Delay Gap | Avg Real Days − Avg Scheduled Days (positive means slower than promised) |
| Quantity Sold | Sum of `Order Item Quantity` |
| Loss Orders % | Share of order lines with negative profit |
| Avg Discount % | Average of `Order Item Discount Rate` |

The dataset has no stock levels, so inventory turnover and stockouts cannot be calculated. `Quantity Sold` is used as a sales-velocity indicator instead.

<details>
<summary>View the DAX measures</summary>

```DAX
Total Sales = SUM(Orders[Sales])

Total Profit = SUM(Orders[Order Profit Per Order])

Orders = DISTINCTCOUNT(Orders[Order Id])

Profit Margin % = DIVIDE([Total Profit], [Total Sales])

Late Delivery % = AVERAGE(Orders[Late_delivery_risk])

Avg Real Days = AVERAGE(Orders[Days for shipping (real)])

Avg Scheduled Days = AVERAGE(Orders[Days for shipment (scheduled)])

Delay Gap = [Avg Real Days] - [Avg Scheduled Days]

Quantity Sold = SUM(Orders[Order Item Quantity])

Loss Orders % =
DIVIDE(
    CALCULATE(COUNTROWS(Orders), Orders[Order Profit Per Order] < 0),
    COUNTROWS(Orders)
)

Avg Discount % = AVERAGE(Orders[Order Item Discount Rate])
```

Calendar table:

```DAX
Calendar =
VAR MinD = DATE(YEAR(MIN(Orders[Order Date])), 1, 1)
VAR MaxD = DATE(YEAR(MAX(Orders[Order Date])), 12, 31)
RETURN
ADDCOLUMNS(
    CALENDAR(MinD, MaxD),
    "Year", YEAR([Date]),
    "Quarter", "Q" & QUARTER([Date]),
    "Month Number", MONTH([Date]),
    "Month", FORMAT([Date], "MMM"),
    "Year-Month", FORMAT([Date], "YYYY-MM")
)
```

</details>

## 7. Dashboard Overview

The report has a landing page and three analysis pages. Navigation buttons on every page let users move between pages. The Year and Market slicers appear on all three analysis pages, with extra slicers where they matter (Department Name, Shipping Mode, and Category Name).

### Welcome Page

A landing page with three navigation buttons: Overview, Delivery, and Products & Profit.

### Page 1 – Overview

- **KPIs:** Total Sales, Total Profit, Profit Margin %, Orders, Late Delivery %.
- **Charts:** monthly sales and profit, sales by customer segment, sales by market, and a map of sales by order country.
- **Slicers:** Market, Year, Department Name.

![Overview Page](images/page1_overview.png)

### Page 2 – Delivery

- **KPIs:** Late Delivery %, Avg Real Days, Avg Scheduled Days, Delay Gap.
- **Charts:** a matrix of Late Delivery % by order region and shipping mode, actual vs. scheduled days by shipping mode, Late Delivery % by shipping mode, and a donut of orders by delivery status.
- **Slicers:** Market, Year, Shipping Mode.

![Delivery Page](images/page2_delivery.png)

### Page 3 – Products & Profit

- **KPIs:** Total Profit, Profit Margin %, Loss Orders %, Avg Discount %.
- **Charts:** profit by category, a product table (quantity sold, sales, profit, margin) sorted to show the least profitable products first, a category matrix with average discount, margin, and loss share, and a sales treemap by category.
- **Slicers:** Year, Market, Department Name, Category Name.

![Products and Profit Page](images/page3_products_profit.png)

## 8. Key Results

### Overall (2015 – 2017)

| KPI | Result |
|---|---|
| Total Sales | 36.47M |
| Total Profit | 3.93M |
| Profit Margin | 10.79% |
| Orders | 64K |
| Late Delivery % | 54.80% |
| Avg Real Days | 3.5 |
| Avg Scheduled Days | 2.9 |
| Delay Gap | 0.6 days |
| Loss Orders % | 18.72% |
| Avg Discount % | 10.17% |

### Late Delivery by Shipping Mode

| Shipping mode | Late Delivery % | Avg real days | Avg scheduled days |
|---|---|---|---|
| First Class | 95.3% | 2.0 | 1.0 |
| Second Class | 76.6% | 4.0 | 2.0 |
| Same Day | 45.8% | 0.5 | 0.0 |
| Standard Class | 38.0% | 4.0 | 4.0 |

Values are rounded as shown in the dashboard.

### Orders by Delivery Status

| Status | Share of orders |
|---|---|
| Late delivery | 54.76% |
| Advance shipping | 23.02% |
| Shipping on time | 17.86% |
| Shipping canceled | 4.36% |

### Sales by Market

| Market | Total Sales |
|---|---|
| Europe | 10.9M |
| LATAM | 10.3M |
| Pacific Asia | 8.0M |
| USCA | 5.1M |
| Africa | 2.3M |

### Sales by Customer Segment

| Segment | Total Sales | Share |
|---|---|---|
| Consumer | 18.94M | 51.94% |
| Corporate | 11.07M | 30.34% |
| Home Office | 6.46M | 17.72% |

### Top Categories by Profit

| Rank | Category | Total Profit |
|---|---|---|
| 1 | Fishing | 756.2K |
| 2 | Cleats | 494.6K |
| 3 | Camping & Hiking | 427.5K |
| 4 | Cardio Equipment | 383.0K |
| 5 | Women's Apparel | 350.4K |

### Examples of Low-Margin Categories

| Category | Profit Margin % | Loss Orders % |
|---|---|---|
| As Seen on TV! | 3.47% | – |
| DVDs | 6.58% | 23.83% |
| Basketball | 6.81% | 16.42% |
| Health and Beauty | 8.30% | 24.33% |

### Examples of High-Margin Categories

| Category | Profit Margin % | Loss Orders % |
|---|---|---|
| Sporting Goods | 17.66% | 10.34% |
| Golf Bags & Carts | 17.46% | 9.84% |
| Toys | 16.59% | 14.53% |

### Products with Negative Total Profit

| Product | Quantity Sold | Total Sales | Total Profit |
|---|---|---|---|
| SOLE E35 Elliptical | 15 | 29,999.85 | -965.12 |
| Bushnell Pro X7 Jolt Slope Rangefinder | 11 | 6,599.89 | -255.95 |
| SOLE E25 Elliptical | 10 | 9,999.90 | -169.56 |

## 9. Key Insights

**More than half of the orders are late.** About 54.8% of orders are flagged as late, and only about 41% arrive early or on time (advance shipping plus on time). This is the largest operational issue in the data.

**Late delivery is a shipping-mode problem, not a region problem.** First Class is late about 95% of the time and Second Class about 77%, while Standard Class is late about 38%. In the region matrix, the First Class column is above 92% in every region shown, and the Standard Class column stays between roughly 28% and 43%. Regional totals sit in a narrow band (most between about 53% and 58%, with Canada lowest at 48.8% among the rows shown), so the region is not what separates good deliveries from bad ones.

**The faster shipping modes make promises they cannot keep.** First Class is scheduled at 1 day but takes about 2 days on average. Second Class is scheduled at 2 days but takes about 4. Standard Class averages the same real and scheduled days (4 and 4), yet is still late 38% of the time because the average hides orders on both sides of the promise. Reducing the gap between the promised and real times for First and Second Class would have the largest effect on late delivery.

**Sales are concentrated in two markets and a few categories.** Europe and LATAM together account for about 58% of sales. Fishing is the largest category at 6.93M (about 19% of total sales) and also the top category by profit (756.2K). Consumers contribute about half of sales (51.94%).

**About 1 in 5 order lines loses money, and the share varies a lot by category.** Loss Orders % is 18.72% overall. In the category matrix it ranges from about 10% (Sporting Goods, Golf Bags & Carts) to about 24% (Health and Beauty, DVDs).

**Categories with higher margins tend to have lower loss shares.** Sporting Goods (17.66% margin) and Golf Bags & Carts (17.46%) have loss shares near 10%, while DVDs (6.58%) and Health and Beauty (8.30%) have loss shares near 24%. This is an observation from the matrix, not proof of cause.

**Discounting does not explain the differences between categories.** The average discount stays between about 9% and 10% across the categories shown, including both the high-margin and low-margin ones. Order-level discounting was not analyzed.

**Only a few products have negative total profit, and they are expensive, low-volume items.** Two SOLE ellipticals and a Bushnell rangefinder are the only products with a negative total profit, each with fewer than 20 units sold.

## 10. Business Recommendations

Based on the findings, I would recommend the following actions:

1. **Reset the promised delivery times for First Class and Second Class.** The real shipping time is about double the scheduled time for both. A realistic promise would reduce late deliveries without changing operations.
2. **Review the carriers and processes behind First Class and Second Class.** If the promise should stay as it is, investigate why these modes miss it so often.
3. **Do not treat late delivery as a regional problem.** Since the regions behave similarly, regional fixes are unlikely to help. Focus on shipping mode.
4. **Track late delivery by shipping mode as a standing KPI.** The overall rate hides large differences between modes.
5. **Investigate loss-making order lines.** About 18.7% of lines lose money. Review costs, prices, and shipping costs on those lines to find the common cause.
6. **Review pricing and costs for the categories with high loss shares and low margins,** such as Health and Beauty, DVDs, As Seen on TV!, and Basketball.
7. **Review the few products with negative total profit,** such as the SOLE ellipticals and the Bushnell rangefinder, and decide whether to reprice or drop them.
8. **Analyze discounting at the order level** before drawing conclusions, because category averages cannot rule it out as a driver of individual losses.
9. **Add stock level data to extend the analysis.** Inventory turnover and stockout analysis need stock data that is not in this dataset.

These recommendations are intended as starting points for further analysis rather than definitive business decisions.

## 11. Limitations

Several limitations should be considered when interpreting the results:

- The dataset has **no stock or inventory levels**, so inventory turnover and stockouts cannot be calculated. This is a supply chain performance analysis, not an inventory analysis.
- 2018 was **excluded** because it contains only a small number of early-year orders.
- Sales are lowest in December (2.6M, compared with roughly 2.9M to 3.2M in the other months). The dataset does not explain why, and the end of the data period may be less complete, so this is not treated as a confirmed seasonal effect.
- **Late Delivery %** uses the dataset's own late-delivery flag. The dashboard's KPI is calculated on order lines, while the delivery status donut counts distinct orders, so the two figures differ slightly (54.80% vs. 54.76%).
- **Loss Orders %** is the share of order lines with negative profit, not the share of whole orders.
- The explanation that the promised times are unrealistic is based on average days. It is a likely explanation, not a confirmed cause.
- The shipping days are recorded as whole days, so averages are approximate.
- Cost details are not available, so the reasons behind the negative-profit lines cannot be confirmed.
- Category-level findings are taken from the category matrix, which lists many categories. The examples cited are not necessarily the highest or lowest overall.
- The data is a historical dataset and may not reflect the company's current operations.
- The map visual is illustrative only. The analysis relies on the bar charts, matrices, and tables.

## 12. Tools Used

- **Power BI Desktop:** Report design, interactivity, navigation, and visuals.
- **Power Query:** Data cleaning, type conversion, and filtering.
- **DAX:** Business measures, KPIs, and the calendar table.
- **Data Modeling:** One fact table, one date table, and a dedicated measures table.
- **Conditional Formatting:** Gradient colors in the matrices and treemap.

## 13. Repository Contents

| File or Folder | Description |
|---|---|
| `DataCo_Dashboard.pbix` | Power BI report containing the data model, measures, and dashboard pages |
| `images/` | Dashboard screenshots, Power Query steps, and data model |
| `README.md` | Project documentation, methodology, results, and recommendations |

To interact with the dashboard, download `DataCo_Dashboard.pbix` and open it in Power BI Desktop on Windows. The raw dataset is not included in this repository because of its size. Download it from Kaggle or Mendeley Data and update the data source path in Power Query if you want to refresh the report.

GitHub does not display Power BI interactivity, so the screenshots in this README provide a visual preview of the analysis.

**Data Source:** DataCo Smart Supply Chain for Big Data Analysis, Kaggle and Mendeley Data.

---

## About the Author

**Ahmed Noby | Aspiring Data Analyst**

Interested in turning raw data into meaningful insights through data analysis, visualization, and business reporting.

Skills demonstrated in this project:

- Power BI: Data Modeling, Interactive Dashboards, Navigation, and Conditional Formatting
- Power Query: Data Cleaning and Transformation
- DAX
- Data Validation
- Data Visualization
- Business Reporting

## Live Power BI Dashboard

[Open the interactive Power BI Dashboard](YOUR_POWER_BI_LINK_HERE)
