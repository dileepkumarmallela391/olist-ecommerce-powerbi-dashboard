
# Olist E-Commerce Sales & Delivery Analytics Dashboard

An interactive 3-page Power BI dashboard analyzing 99,000+ orders from Olist, a Brazilian e-commerce marketplace, covering sales performance, delivery reliability, and their measurable impact on customer satisfaction.

## Dashboard Preview

![Executive Overview](Screenshot%202026-08-11%20000632.png)
![Delivery Performance](Screenshot%202026-08-11%20000653.png)
![Customer & Payment Insights](Screenshot%202026-08-11%20000729.png)

---

## Business Problem

E-commerce businesses depend on two things to retain customers: competitive products and reliable delivery. This project set out to answer:

- Which product categories and states drive the most revenue?
- How reliable is Olist's delivery performance, and where does it break down?
- Does delivery performance actually affect customer satisfaction — and by how much?
- What payment behaviors are most common among customers?

## Key Findings

1. **Delivery delays significantly hurt customer satisfaction.**
   Orders delivered late received an average review score of **2.3**, compared to **4.3** for on-time orders — a 2-point drop. This is the strongest satisfaction driver identified in the dataset, stronger than product category or payment method.

2. **Overall delivery performance is strong.**
   90.45% of orders were delivered on time, and on average, orders arrived about 12 days *ahead* of the estimated delivery date — suggesting Olist sets conservative delivery estimates rather than an efficiency problem.

3. **Late deliveries are concentrated in high-volume states.**
   São Paulo (SP) and Rio de Janeiro (RJ) account for both the highest revenue and the highest number of late orders, which tracks with their share of total order volume rather than indicating a regional service problem.

4. **Credit card is the dominant payment method**, accounting for 78% of total revenue, followed by boleto (17%). Voucher and debit card payments make up a small remaining share.

5. **Health & Beauty and Watches/Gifts are the top revenue-generating categories**, followed by Bed/Bath/Table and Sports/Leisure.

6. **Payment installment count has minimal effect on review scores** — average scores stayed relatively stable (mostly 3.5–4.5) across most installment counts, with outliers at very high installment counts likely due to small sample sizes.

## Dataset

**[Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)** (Kaggle) — ~100,000 orders placed on the Olist marketplace between 2016–2018, split across 9 relational CSV files:

| Table | Contents |
|---|---|
| `orders` | Order status, timestamps (purchase, approved, delivered, estimated delivery) |
| `order_items` | Product, price, freight value per order line item |
| `order_payments` | Payment type, installments, payment value |
| `order_reviews` | Review score, review comments |
| `customers` | Customer ID, city, state |
| `products` | Product category, dimensions, weight |
| `sellers` | Seller ID, city, state |
| `product_category_name_translation` | Portuguese → English category names |
| `geolocation` | Zip code to lat/lng mapping |

## Data Modeling

Built a star-schema data model connecting all 9 tables through their natural keys:

- `orders[customer_id]` → `customers[customer_id]`
- `order_items[order_id]` → `orders[order_id]`
- `order_items[product_id]` → `products[product_id]`
- `order_items[seller_id]` → `sellers[seller_id]`
- `order_payments[order_id]` → `orders[order_id]` *(cross-filter direction set to Both — required for payment-type-level revenue slicing, since order_items sits on the opposite side of the orders table from payments)*
- `order_reviews[order_id]` → `orders[order_id]`
- `products[product_category_name]` → `product_category_name_translation[product_category_name]`

A dedicated **Date table** was built using `CALENDAR()`, marked as an official date table, and connected via a cleaned `Order Date` column (derived from `order_purchase_timestamp` with `INT()` to strip the time component, since the raw timestamp's time portion broke the relationship's row matching).

## DAX Measures & Calculated Columns

**Core measures:**
```dax
Total Revenue = SUM(olist_order_items_dataset[price])
Total Orders = DISTINCTCOUNT(olist_orders_dataset[order_id])
Avg Order Value = DIVIDE([Total Revenue], [Total Orders])
Avg Review Score = AVERAGE(olist_order_reviews_dataset[review_score])
```

**Delivery analysis:**
```dax
Delivery Variance Days = 
DATEDIFF(
    olist_orders_dataset[order_estimated_delivery_date],
    olist_orders_dataset[order_delivered_customer_date],
    DAY
)

Delivery Status = 
IF(
    ISBLANK(olist_orders_dataset[order_delivered_customer_date]),
    "Not Delivered",
    IF(olist_orders_dataset[Delivery Variance Days] > 0, "Late", "On Time")
)

On-Time Delivery % = 
DIVIDE(
    CALCULATE([Total Orders], olist_orders_dataset[Delivery Status] = "On Time"),
    [Total Orders]
)
```

**The headline insight — conditional review scores by delivery status:**
```dax
Avg Review Score (Late Orders) = 
CALCULATE([Avg Review Score], olist_orders_dataset[Delivery Status] = "Late")

Avg Review Score (On-Time Orders) = 
CALCULATE([Avg Review Score], olist_orders_dataset[Delivery Status] = "On Time")
```

**Time intelligence:**
```dax
Year Month Number = YEAR(DateTable[Date]) * 100 + MONTH(DateTable[Date])
```
Used as a hidden sort key so `Year Month` (e.g., "Jan 2017") displays in true chronological order instead of Power BI's default alphabetical sort.

## Dashboard Pages

**1. Executive Overview**
KPI cards (Total Revenue, Total Orders, Avg Order Value, Avg Review Score), a monthly revenue trend line, and a Top 10 product category bar chart.

**2. Delivery Performance**
KPI cards (On-Time Delivery %, Avg Delivery Delay, Late Orders), the featured On-Time vs. Late review score comparison chart, an on-time delivery % trend over time, and late orders broken down by state.

**3. Customer & Payment Insights**
Review score distribution, average review score by payment installment count, revenue by payment type (donut), and revenue by customer state.

## Build Process & Challenges

This project was built end-to-end in Power BI Desktop, including troubleshooting real data-modeling issues rather than working from a pre-cleaned dataset:

- **Relationship debugging:** a revenue-by-month line chart initially returned a flat, incorrect total for every month because the relationship between the Date table and orders was joined on a raw datetime column with a time component, which silently broke row matching. Fixed by deriving a clean date-only column and rejoining.
- **Cross-filter direction:** revenue by payment type initially returned an identical value split evenly across all payment types, because the default single-direction relationship didn't allow filters from the payments table to reach the order_items table where revenue lives. Fixed by setting the relationship to bidirectional filtering.
- **Column vs. measure errors:** several DAX formulas (e.g., delivery variance, sort-key columns) initially failed because they were created as measures instead of calculated columns — a common distinction between row-level calculations and aggregations in DAX.
- **Custom sort order:** month labels like "Jan 2017" sort alphabetically by default; fixed using a numeric `Year Month Number` column and Power BI's "Sort by column" feature.

## Tools & Skills Used

- **Power BI Desktop** — Power Query (data cleaning & transformation), DAX (measures & calculated columns), data modeling (star schema, relationships, cross-filter direction)
- **Data visualization** — KPI cards, line charts, bar charts, donut charts, custom color theming, page alignment/formatting
- **Analytical thinking** — identifying a non-obvious business insight (delivery delay's effect on satisfaction) beyond basic revenue reporting

## Notes

- The sharp revenue drop visible in September 2018 reflects the dataset's cutoff date, not an actual business decline.
- All monetary values are in Brazilian Real (R$).
