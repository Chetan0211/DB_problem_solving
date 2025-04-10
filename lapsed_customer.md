# Problem: Identifying High-Value Lapsed Customers for a Re-engagement Campaign 

### Scenario: 

You are working for an e-commerce company. The marketing team wants to launch a targeted re-engagement campaign aimed at customers who were previously highly valuable but haven't made a purchase in a significant amount of time. Your task is to generate a list of these "High-Value Lapsed Customers" using the company's PostgreSQL database. 

### Database Schema (Simplified): 

Assume you have the following relevant tables: 

#### 1. customers 

    customer_id (INTEGER, Primary Key) 

    name (VARCHAR) 

    email (VARCHAR, Unique) 

    registration_date (TIMESTAMP) 

#### 2. products 

    product_id (INTEGER, Primary Key) 

    name (VARCHAR) 

    category (VARCHAR) 

    unit_price (DECIMAL(10, 2)) -- Price at the time it was listed, might change. 

#### 3. orders 
    order_id (INTEGER, Primary Key) 

    customer_id (INTEGER, Foreign Key referencing customers) 

    order_date (TIMESTAMP) 

    status (VARCHAR) -- e.g., 'completed', 'shipped', 'pending', 'cancelled' 

#### 4. order_items 

    order_item_id (INTEGER, Primary Key) 

    order_id (INTEGER, Foreign Key referencing orders) 

    product_id (INTEGER, Foreign Key referencing products) 

    quantity (INTEGER) 

    price_at_purchase (DECIMAL(10, 2)) -- The actual price paid per unit for this item in this order. 

 ### Task: 

Write a single SQL query for PostgreSQL to identify customers who meet both of the following criteria: 

1. **High-Value**: Customers whose total lifetime spending ranks in the top 10% of all customers based on their spending. 

Lifetime spending is calculated as the sum of (quantity * price_at_purchase) for all items across all their 'completed' orders. 

Customers with zero completed orders should not be considered in the top 10% calculation or the final list. 

2. **Lapsed**: Customers whose most recent 'completed' order was placed more than 6 months ago from a specific reference date. For this problem, use '2025-04-10' as the reference date. 

### Required Output: 

The query should return the following columns for each identified customer: 

* customer_id 

* customer_name (from the customers table) 

* customer_email (from the customers table) 

* total_lifetime_spending (calculated total spending for completed orders) 

* last_order_date (the timestamp of their most recent 'completed' order) 

### Considerations & Potential Challenges: 

* Calculating total spending per customer accurately, considering only 'completed' orders. 

* Determining the date of the last completed order for each customer. 

* Implementing the "top 10%" logic. PostgreSQL offers several ways to handle rankings and percentiles (e.g., Window Functions like NTILE, RANK, DENSE_RANK, or using counts and PERCENTILE_CONT/PERCENTILE_DISC - choose what you think is best). 

* Efficiently joining the tables. 

* Handling date comparisons and intervals correctly in PostgreSQL (e.g., using INTERVAL '6 months'). 

* Customers who have placed orders, but none were 'completed'. 

* Performance on potentially large tables (though you don't need to worry about creating indexes for this exercise, think about how the query might perform). 

### Deliverable: 

Your final PostgreSQL query. You can also add brief comments within the SQL or notes explaining your approach, especially regarding the "top 10%" calculation method. 

 
 

Good luck! Take your time, think through the steps, and feel free to ask for clarification on any part of the problem description. Let me know when you have a solution ready for review. 

 


## Solution:

1. Creating tables
```sql
-- Create pgcrypto for UUID generation
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Create customer table
CREATE TABLE IF NOT EXISTS lapsed_customer_problem.customers(
	id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	name character varying not null,
	gmail character varying not null unique,
	created_at timestamp without time zone not null,
	updated_at timestamp without time zone not null
);

-- Create categories table
CREATE TABLE IF NOT EXISTS lapsed_customer_problem.categories(
	id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	name character varying not null,
	value character varying not null,
	created_at timestamp without time zone not null,
	updated_at timestamp without time zone not null
);

-- Create index for category value
CREATE INDEX idx_categories_value ON lapsed_customer_problem.categories(value);

-- Create products table
CREATE TABLE IF NOT EXISTS lapsed_customer_problem.products(
	id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	name character varying not null,
	category_id uuid,
	unit_prize NUMERIC(10,2),
	created_at timestamp without time zone not null,
	updated_at timestamp without time zone not null,

	CONSTRAINT fk_products_category_id
		FOREIGN KEY (category_id) REFERENCES lapsed_customer_problem.categories(id)
		ON DELETE SET NULL
		ON UPDATE CASCADE
);

-- Create index for products category_id
CREATE INDEX idx_products_category_id ON lapsed_customer_problem.products(category_id);

-- Create orders table
CREATE TABLE IF NOT EXISTS lapsed_customer_problem.orders(
	id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	customer_id uuid,
	status character varying not null,
	created_at timestamp without time zone not null,
	updated_at timestamp without time zone not null,

	CONSTRAINT fk_orders_customer_id
		FOREIGN KEY (customer_id) REFERENCES lapsed_customer_problem.customers(id)
		ON DELETE SET NULL
		ON UPDATE CASCADE,
	CONSTRAINT status_check CHECK(status IN ('completed', 'shipped', 'pending', 'ordered', 'cancelled'))
);

-- Create index for orders customer_id
CREATE INDEX idx_orders_customer_id ON lapsed_customer_problem.orders(customer_id);

-- Create order_items table
CREATE TABLE IF NOT EXISTS lapsed_customer_problem.order_items(
	id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	order_id uuid,
	product_id uuid,
	quantity int,
	total_product_price NUMERIC(10,2),

	CONSTRAINT fk_order_items_order_id
		FOREIGN KEY (order_id) REFERENCES lapsed_customer_problem.orders(id)
		ON DELETE SET NULL
		ON UPDATE CASCADE,
	CONSTRAINT fk_order_items_product_id
		FOREIGN KEY (product_id) REFERENCES lapsed_customer_problem.products(id)
		ON DELETE SET NULL
		ON UPDATE CASCADE
);

-- Create indexes for order_items order_id and product_id
CREATE INDEX idx_order_items_order_id ON lapsed_customer_problem.order_items(order_id);
CREATE INDEX idx_order_items_product_id ON lapsed_customer_problem.order_items(product_id);
```
