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
* **File:** `problem_1.sql`

### 2. Demand Seasonality & Trend Analysis

**Objective:** Track product performance across months and years.

* Calculates monthly sales percentages to identify seasonal peaks.
* Computes YoY growth percentages for product categories using window functions (`LAG`).
* **File:** `problem_2.sql`

### 3. Geospatial Micro-Warehousing

**Objective:** Reduce "Last Mile" costs by identifying demand hotspots.

* Clusters customers by rounding latitude and longitude to one decimal point.
* Identifies the top 5 high-volume areas and top 3 high-shipping-cost areas for potential fulfillment centers.
* **File:** `problem_3.sql`

### 4. Seller Risk & Churn Prediction

**Objective:** Identify sellers whose logistics failures are driving away new customers.

* **Criteria**: Orders with  days of delay and a review score  from first-time customers.
* **Risk Flag**: Sellers with  churn probability are flagged as `HIGH RISK`.
* **File:** `problem_4.sql`

### 5. Collaborative Shipping (Lost Margin Analysis)

**Objective:** Find opportunities for sellers to co-locate inventory.

* Analyzes orders where products from different sellers are bought together.
* Calculates "Lost Margin"—the extra shipping cost incurred by sending two separate packages instead of one consolidated shipment.
* **File:** `problem_5.sql`

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
