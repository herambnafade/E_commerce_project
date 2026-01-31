# E_commerce_project
An advanced SQL analytics framework for e-commerce supply chain optimization. It utilizes PostgreSQL (CTEs, Window Functions) to calculate EOQ and Reorder Points, identify geospatial delivery clusters, and flag high-risk sellers driving customer churn. Features monthly seasonality and YoY growth tracking.

# E-commerce Supply Chain & Logistics Analytics

This repository contains a comprehensive SQL-based analysis of an e-commerce ecosystem. The project focuses on solving complex operational challenges such as **inventory optimization**, **demand forecasting**, **geospatial logistics**, and **seller risk management**.

---

## 📊 Project Overview

The analysis uses a relational database schema consisting of customers, orders, products, sellers, and geospatial data. The goal is to derive actionable insights to reduce shipping costs, improve delivery times, and optimize stock levels.

### Key Analytical Pillars:

* **Inventory Management:** Implementing Economic Order Quantity (EOQ) and Reorder Point (ROP) models.
* **Demand Intelligence:** Seasonality analysis and Year-over-Year (YoY) growth tracking.
* **Logistics Optimization:** Identifying high-demand geospatial clusters for micro-warehousing.
* **Risk Mitigation:** Identifying high-risk sellers based on delivery delays and customer churn.
* **Collaborative Logistics:** Analyzing co-purchasing patterns to suggest seller collaborations.

---

## 🗄️ Database Schema

The project utilizes eight primary tables. The schema is designed to track the entire lifecycle of an order from purchase to delivery and review.

* **`customers`**: Unique identifiers and location data for buyers.
* **`orders`**: Timestamps for purchase, approval, and delivery.
* **`products`**: Product IDs and category names.
* **`sellers`**: Seller IDs and location data.
* **`order_items`**: Pricing, freight value, and shipping limits.
* **`order_payments`**: Transaction details and payment types.
* **`order_reviews`**: Customer satisfaction scores.
* **`geolocation`**: Latitudinal and longitudinal data by zip code.
```sql
CREATE TABLE customers (
    customer_id VARCHAR(50) PRIMARY KEY,
    customer_unique_id VARCHAR(50),
    customer_zip_code_prefix INT,
    customer_city VARCHAR(50),
    customer_state VARCHAR(10)
);

CREATE TABLE orders (
    order_id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50),
    order_status VARCHAR(50),
    order_purchase_timestamp TIMESTAMP,
    order_approved_at TIMESTAMP,
    order_delivered_carrier_date TIMESTAMP,
    order_delivered_customer_date TIMESTAMP,
    order_estimated_delivery_date TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE products (
    product_id VARCHAR(50) PRIMARY KEY,
    product_category_name VARCHAR(50),
    
);

CREATE TABLE sellers (
    seller_id VARCHAR(50) PRIMARY KEY,
    seller_zip_code_prefix INT,
    seller_city VARCHAR(50),
    seller_state VARCHAR(10)
);

CREATE TABLE order_items (
    order_id VARCHAR(50),
    order_item_id INT,
    product_id VARCHAR(50),
    seller_id VARCHAR(50),
    shipping_limit_date TIMESTAMP,
    price FLOAT,
    freight_value FLOAT,
    PRIMARY KEY (order_id, order_item_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (seller_id) REFERENCES sellers(seller_id)
);

CREATE TABLE order_payments (
    order_id VARCHAR(50),
    payment_sequential INT,
    payment_type VARCHAR(20),
    payment_value FLOAT,
    PRIMARY KEY (order_id, payment_sequential),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);


CREATE TABLE order_reviews (
    review_id VARCHAR(50),
    order_id VARCHAR(50),
    review_score INT,
    review_creation_date TIMESTAMP,
    review_answer_timestamp TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    PRIMARY KEY (review_id, order_id)
);

CREATE TABLE geolocation (
    geolocation_zip_code_prefix INT,
    geolocation_lat FLOAT,
    geolocation_lng FLOAT,	
    geolocation_city VARCHAR(50),
    geolocation_state VARCHAR(10)
);

```
---

## 🚀 Problem Statements & Solutions

### 1. Inventory Optimization (EOQ & ROP)

**Objective:** Determine how much to order and when to order to minimize holding and ordering costs.

* **EOQ Formula**: Uses annual demand, freight value (as ordering cost), and 20% of price as holding cost.
* **Reorder Point**: Accounts for average lead time and a 95% service level safety stock ().
* ```sql
  SELECT 
    product_id,
    ROUND(annual_demand::numeric, 2) AS annual_demand,
    ROUND(eoq::numeric, 0) AS eoq,
    ROUND((1.65 * lead_time_sigma * (annual_demand / 365.0))::numeric, 0) AS safety_stock,    --Safety Stock calculation: Z (1.65 for 95%) * σLT * (Annual Demand / 365)
    ROUND(((avg_lead_time * (annual_demand / 365.0)) + (1.65 * lead_time_sigma * (annual_demand / 365.0)))::numeric, 0) AS reorder_point    --Reorder Point = (Avg Lead Time * Daily Demand) + Safety Stock
FROM (
    SELECT 
        oi.product_id,
        AVG(oi.price) AS avg_price,
        AVG(oi.freight_value) AS avg_freight,
        COUNT(oi.order_id) / NULLIF((MAX(o.order_purchase_timestamp)::date - MIN(o.order_purchase_timestamp)::date) / 365.0, 0) AS annual_demand,      -- EOQ Components
        AVG(EXTRACT(DAY FROM (o.order_delivered_customer_date - o.order_purchase_timestamp))) AS avg_lead_time,      -- Lead Time Components (in days)
        STDDEV(EXTRACT(DAY FROM (o.order_delivered_customer_date - o.order_purchase_timestamp))) AS lead_time_sigma,
        SQRT((2 * (COUNT(oi.order_id) / NULLIF((MAX(o.order_purchase_timestamp)::date - MIN(o.order_purchase_timestamp)::date) / 365.0, 0)) * AVG(oi.freight_value)) / (AVG(oi.price) * 0.2)) AS eoq   -- EOQ Formula
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.order_status = 'delivered' 
      AND o.order_delivered_customer_date IS NOT NULL
    GROUP BY oi.product_id
    HAVING COUNT(oi.order_id) > 5
) AS stats
WHERE avg_price > 0
ORDER BY reorder_point DESC;
```

### 2. Demand Seasonality & Trend Analysis

**Objective:** Track product performance across months and years.

* Calculates monthly sales percentages to identify seasonal peaks.
* Computes YoY growth percentages for product categories using window functions (`LAG`).
* ```sql
SELECT
    p.product_id,
    p.stock_quantity AS current_stock,
    EXTRACT(YEAR FROM o.order_purchase_timestamp) AS distinct_year,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 1) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Jan_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 2) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Feb_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 3) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Mar_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 4) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Apr_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 5) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS May_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 6) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Jun_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 7) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Jul_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 8) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Aug_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 9) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Sep_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 10) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Oct_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 11) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Nov_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 12) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Dec_Pct,
    COUNT(oi.order_item_id) AS total_yearly_units
FROM products AS p
JOIN order_items AS oi ON p.product_id = oi.product_id
JOIN orders AS o ON oi.order_id = o.order_id
GROUP BY 
    p.product_id, 
    p.stock_quantity, 
    distinct_year
ORDER BY 
    distinct_year DESC, 
    total_yearly_units DESC;

1
SELECT
    p.product_category_name,
    EXTRACT(YEAR FROM o.order_purchase_timestamp) AS distinct_year,
    COUNT(oi.order_item_id) AS total_yearly_units,
    ROUND(
        100.0 * (COUNT(oi.order_item_id) - LAG(COUNT(oi.order_item_id)) OVER (
            PARTITION BY p.product_category_name 
            ORDER BY EXTRACT(YEAR FROM o.order_purchase_timestamp)
        )) / NULLIF(LAG(COUNT(oi.order_item_id)) OVER (
            PARTITION BY p.product_category_name 
            ORDER BY EXTRACT(YEAR FROM o.order_purchase_timestamp)
        ), 0), 
    2) AS yoy_growth_pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 1) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Jan_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 2) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Feb_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 3) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Mar_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 4) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Apr_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 5) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS May_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 6) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Jun_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 7) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Jul_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 8) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Aug_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 9) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Sep_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 10) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Oct_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 11) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Nov_Pct,
    ROUND(100.0 * COUNT(oi.order_item_id) FILTER (WHERE EXTRACT(MONTH FROM o.order_purchase_timestamp) = 12) / NULLIF(COUNT(oi.order_item_id), 0), 2) AS Dec_Pct
FROM products AS p
JOIN order_items AS oi ON p.product_id = oi.product_id
JOIN orders AS o ON oi.order_id = o.order_id
GROUP BY 
    p.product_category_name,
    distinct_year
ORDER BY 
    p.product_category_name, 
    distinct_year DESC;
```

### 3. Geospatial Micro-Warehousing

**Objective:** Reduce "Last Mile" costs by identifying demand hotspots.

* Clusters customers by rounding latitude and longitude to one decimal point.
* Identifies the top 5 high-volume areas and top 3 high-shipping-cost areas for potential fulfillment centers.
* ```sql
CREATE TEMP TABLE demand_clusters AS
WITH customer_locations AS (
    SELECT
        o.order_id AS order_id,
        oi.freight_value AS freight_value,
        g.geolocation_lat AS customer_latitude,
        g.geolocation_lng AS customer_longitude
    FROM orders o
    INNER JOIN order_items oi
        ON o.order_id = oi.order_id
    INNER JOIN customers c
        ON o.customer_id = c.customer_id
    INNER JOIN geolocation g
        ON c.customer_zip_code_prefix = g.geolocation_zip_code_prefix
), demand_clusters AS (
    SELECT
        ROUND(customer_latitude::numeric, 1) AS cluster_latitude,
        ROUND(customer_longitude::numeric, 1) AS cluster_longitude,
        COUNT(order_id) AS total_orders,
        SUM(freight_value) AS total_shipping_cost
    FROM customer_locations
    GROUP BY
        ROUND(customer_latitude::numeric, 1),
        ROUND(customer_longitude::numeric, 1)
)

-- Top 5 high-demand clusters (by order volume)
SELECT *
FROM demand_clusters
ORDER BY total_orders DESC
LIMIT 5;

-- micro-fulfillment centers (by shipping cost)
SELECT *
FROM demand_clusters
ORDER BY total_shipping_cost DESC
LIMIT 3;
```

### 4. Seller Risk & Churn Prediction

**Objective:** Identify sellers whose logistics failures are driving away new customers.

* **Criteria**: Orders with  days of delay and a review score  from first-time customers.
* **Risk Flag**: Sellers with  churn probability are flagged as `HIGH RISK`.
* ```sql
with shipping_data as (
    select
        o.order_id,
        oi.seller_id,
        o.customer_id,
        r.review_score,
        -- calculating raw delay days
        (o.order_delivered_carrier_date::date - oi.shipping_limit_date::date) as delay_days
    from orders o
    join order_items oi on o.order_id = oi.order_id
    join order_reviews r on o.order_id = r.order_id
    where o.order_delivered_carrier_date is not null
),

customer_history as (
    select 
        customer_id, 
        count(order_id) as total_orders
    from orders
    group by customer_id
)

select
    s.seller_id,
    count(s.order_id) as delayed_orders,
    sum(case when h.total_orders = 1 and s.review_score <= 2 then 1 else 0 end) as churned_orders,
    round(
        sum(case when h.total_orders = 1 and s.review_score <= 2 then 1 else 0 end)::numeric 
        / count(s.order_id), 2
    ) as churn_probability,
    case 
        when (sum(case when h.total_orders = 1 and s.review_score <= 2 then 1 else 0 end)::float / count(s.order_id)) > 0.30 
        then 'HIGH RISK'
        else 'LOW RISK'
    end as seller_risk_flag
from shipping_data s
join customer_history h on s.customer_id = h.customer_id
where s.delay_days >= 5
group by s.seller_id
order by churn_probability desc;
```

### 5. Collaborative Shipping (Lost Margin Analysis)

**Objective:** Find opportunities for sellers to co-locate inventory.

* Analyzes orders where products from different sellers are bought together.
* Calculates "Lost Margin"—the extra shipping cost incurred by sending two separate packages instead of one consolidated shipment.
* ```sql
  -- Problem Statement 5A:
-- Seller pairs whose products are bought together (Lost Margin from shipping)
-- Sellers who should collaborate for inventory

SELECT 
    a.seller_id AS seller_a, 
    b.seller_id AS seller_b,
    COUNT(*) AS co_purchase_count,
    ROUND(SUM((a.freight_value + b.freight_value) - GREATEST(a.freight_value, b.freight_value))::numeric, 2) AS total_lost_margin,
    ROUND(AVG((a.freight_value + b.freight_value) - GREATEST(a.freight_value, b.freight_value))::numeric, 2) AS avg_lost_margin
FROM order_items a
JOIN order_items b ON a.order_id = b.order_id 
    AND a.product_id < b.product_id
WHERE a.seller_id <> b.seller_id
GROUP BY seller_a, seller_b
HAVING COUNT(*) >= 10
ORDER BY total_lost_margin DESC;

-- Problem Statement 5B:
-- Products Based on Category which are bought together (Lost Margin from shipping)

SELECT 
    p1.product_category_name AS category_a,
    p2.product_category_name AS category_b,
    COUNT(*) AS co_purchase_count,
    ROUND(SUM((a.freight_value + b.freight_value) - GREATEST(a.freight_value, b.freight_value))::numeric, 2) AS total_lost_margin,
    ROUND(AVG((a.freight_value + b.freight_value) - GREATEST(a.freight_value, b.freight_value))::numeric, 2) AS avg_lost_margin
FROM order_items a
JOIN order_items b 
    ON a.order_id = b.order_id
   AND a.product_id < b.product_id
JOIN products p1 
    ON a.product_id = p1.product_id
JOIN products p2 
    ON b.product_id = p2.product_id
WHERE a.seller_id <> b.seller_id
  AND p1.product_category_name <> p2.product_category_name
GROUP BY 
    p1.product_category_name,
    p2.product_category_name
HAVING COUNT(*) >= 10
ORDER BY co_purchase_count DESC;
```

---

## 🛠️ Key SQL Techniques Used

* **Window Functions**: `LAG()` and `OVER(PARTITION BY...)` for growth trends.
* **CTEs & Temp Tables**: `WITH` clauses and `CREATE TEMP TABLE` for modular logic.
* **Aggregations**: `FILTER (WHERE ...)` and `CASE WHEN` for conditional metrics.
* **Statistical Modeling**: `STDDEV`, `SQRT`, and `AVG` for supply chain calculations.

---

## 📂 Repository Structure

```bash
├── tables.sql          # Schema definitions and DDL
├── load_data.sql       # Bulk data loading scripts (COPY commands)
├── problem_1.sql       # EOQ and Reorder Point logic
├── problem_2.sql       # Seasonality and Trend analysis
├── problem_3.sql       # Geospatial clustering
├── problem_4.sql       # Seller risk and churn analysis
└── problem_5.sql       # Co-purchase and margin analysis

```

---

## ⚙️ How to Run

1. **Initialize Schema**: Execute `tables.sql` to create the database structure.
2. **Import Data**: Update the file paths in `load_data.sql` to point to your local CSV files and run it.
3. **Run Analytics**: Execute any of the `problem_x.sql` files to generate insights for that specific business domain.

Would you like me to create a summary of the data insights or visualizations you could build based on these queries?
