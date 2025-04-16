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

-- Create a funtion to update the updated_at on all tables
CREATE OR REPLACE FUNCTION trigger_set_timestamp()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = CURRENT_TIMESTAMP; -- Or NOW()
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Alter customers table to have default created at and updated at trigger
ALTER TABLE IF EXISTS lapsed_customer_problem.customers
ALTER COLUMN created_at SET DEFAULT CURRENT_TIMESTAMP;

ALTER TABLE IF EXISTS lapsed_customer_problem.customers
ALTER COLUMN updated_at SET DEFAULT CURRENT_TIMESTAMP;

CREATE TRIGGER set_customers_updated_at_timestamp_after_update
AFTER UPDATE ON lapsed_customer_problem.customers
FOR EACH ROW 
EXECUTE FUNCTION trigger_set_timestamp();

-- Alter categories table to have default created at and updated at trigger
ALTER TABLE IF EXISTS lapsed_customer_problem.categories
ALTER COLUMN created_at SET DEFAULT CURRENT_TIMESTAMP;

ALTER TABLE IF EXISTS lapsed_customer_problem.categories
ALTER COLUMN updated_at SET DEFAULT CURRENT_TIMESTAMP;

CREATE TRIGGER set_categories_updated_at_timestamp_after_update
AFTER UPDATE ON lapsed_customer_problem.categories
FOR EACH ROW
EXECUTE FUNCTION trigger_set_timestamp();

-- Alter products table to have default created at and updated at trigger
ALTER TABLE IF EXISTS lapsed_customer_problem.products
ALTER COLUMN created_at SET DEFAULT CURRENT_TIMESTAMP;

ALTER TABLE IF EXISTS lapsed_customer_problem.products
ALTER COLUMN updated_at SET DEFAULT CURRENT_TIMESTAMP;

CREATE TRIGGER set_products_updated_at_timestamp_after_update
AFTER UPDATE ON lapsed_customer_problem.products
FOR EACH ROW
EXECUTE FUNCTION trigger_set_timestamp();

-- Alter orders table to have default created at and updated at trigger
ALTER TABLE IF EXISTS lapsed_customer_problem.orders
ALTER COLUMN created_at SET DEFAULT CURRENT_TIMESTAMP;

ALTER TABLE IF EXISTS lapsed_customer_problem.orders
ALTER COLUMN updated_at SET DEFAULT CURRENT_TIMESTAMP;

CREATE TRIGGER set_orders_updated_at_timestamp_after_update
AFTER UPDATE ON lapsed_customer_problem.orders
FOR EACH ROW
EXECUTE FUNCTION trigger_set_timestamp();

-- Alter order_items table to have default created at and updated at trigger
ALTER TABLE IF EXISTS lapsed_customer_problem.order_items
ADD COLUMN created_at timestamp without time zone not null DEFAULT CURRENT_TIMESTAMP;

ALTER TABLE IF EXISTS lapsed_customer_problem.order_items
ADD COLUMN updated_at timestamp without time zone not null DEFAULT CURRENT_TIMESTAMP;

CREATE TRIGGER set_order_items_updated_at_timestamp_after_update
AFTER UPDATE ON lapsed_customer_problem.order_items
FOR EACH ROW
EXECUTE FUNCTION trigger_set_timestamp();
```

2. Populating the data
```sql
-- Add values into categories table
INSERT INTO lapsed_customer_problem.categories (name, value) VALUES
('Electronics', 'electronics'),
('Clothing', 'clothing'),
('Home & Garden', 'home_garden'),
('Books', 'books'),
('Sports & Outdoors', 'sports_outdoors'),
('Toys & Games', 'toys_games'),
('Beauty & Health', 'beauty_health'),
('Automotive', 'automotive'),
('Computers', 'computers'),
('Mobile Phones', 'mobile_phones'),
('Fashion Accessories', 'fashion_accessories'),
('Furniture', 'furniture'),
('Groceries', 'groceries'),
('Music & Movies', 'music_movies'),
('Office Supplies', 'office_supplies');

-- Add values into products table
INSERT INTO lapsed_customer_problem.products (name, category_id, unit_prize) VALUES
-- Electronics (a0456dcf-1f5a-4b11-a8a7-e60b2e62af1f)
('Wireless Noise-Cancelling Headphones', 'a0456dcf-1f5a-4b11-a8a7-e60b2e62af1f', 199.99),
('Smart LED TV 55-Inch 4K UHD', 'a0456dcf-1f5a-4b11-a8a7-e60b2e62af1f', 479.50),
('Bluetooth Portable Speaker Waterproof', 'a0456dcf-1f5a-4b11-a8a7-e60b2e62af1f', 49.95),
('USB-C Fast Charging Cable 6ft', 'a0456dcf-1f5a-4b11-a8a7-e60b2e62af1f', 12.99),

-- Clothing (582d39d5-d6c6-409b-8f8e-6f8eca1dea9a)
('Men''s Classic Cotton T-Shirt (Navy)', '582d39d5-d6c6-409b-8f8e-6f8eca1dea9a', 19.95),
('Women''s High-Waisted Skinny Jeans', '582d39d5-d6c6-409b-8f8e-6f8eca1dea9a', 44.99),
('Summer Floral Print Maxi Dress', '582d39d5-d6c6-409b-8f8e-6f8eca1dea9a', 59.90),
('Unisex Pullover Hoodie (Grey)', '582d39d5-d6c6-409b-8f8e-6f8eca1dea9a', 35.50),

-- Home & Garden (05ef43fe-445d-4fc9-8845-852b97b1da52)
('Cordless Power Drill Set 20V', '05ef43fe-445d-4fc9-8845-852b97b1da52', 89.99),
('Expandable Garden Hose 50ft', '05ef43fe-445d-4fc9-8845-852b97b1da52', 29.95),
('Set of 4 Ceramic Coffee Mugs', '05ef43fe-445d-4fc9-8845-852b97b1da52', 18.75),
('Outdoor Solar Pathway Lights (8-Pack)', '05ef43fe-445d-4fc9-8845-852b97b1da52', 34.99),

-- Books (7743bcab-f576-44e0-8ac3-1dc1024c63a8)
('The Midnight Library - Hardcover', '7743bcab-f576-44e0-8ac3-1dc1024c63a8', 22.50),
('Atomic Habits - Paperback', '7743bcab-f576-44e0-8ac3-1dc1024c63a8', 16.99),
('Introduction to Python Programming', '7743bcab-f576-44e0-8ac3-1dc1024c63a8', 45.00),
('Children''s Illustrated Storybook', '7743bcab-f576-44e0-8ac3-1dc1024c63a8', 12.95),

-- Sports & Outdoors (2b25a05c-6108-471b-b450-5118a4024e90)
('Yoga Mat Non-Slip (Purple)', '2b25a05c-6108-471b-b450-5118a4024e90', 24.99),
('Adjustable Dumbbell Set (5-25 lbs)', '2b25a05c-6108-471b-b450-5118a4024e90', 129.00),
('Camping Tent 2-Person Waterproof', '2b25a05c-6108-471b-b450-5118a4024e90', 65.50),
('Insulated Stainless Steel Water Bottle 32oz', '2b25a05c-6108-471b-b450-5118a4024e90', 19.99),

-- Toys & Games (3cce9173-13a2-44aa-ad8d-b279795479fb)
('Building Blocks Set (1000 Pieces)', '3cce9173-13a2-44aa-ad8d-b279795479fb', 39.99),
('Remote Control Car Off-Road', '3cce9173-13a2-44aa-ad8d-b279795479fb', 29.95),
('Strategy Board Game - Settlers of Catan', '3cce9173-13a2-44aa-ad8d-b279795479fb', 44.50),
('Plush Teddy Bear (Large)', '3cce9173-13a2-44aa-ad8d-b279795479fb', 22.00),

-- Beauty & Health (3e97d950-b115-4a2f-a737-daa0a9c9fbb8)
('Vitamin C Serum for Face', '3e97d950-b115-4a2f-a737-daa0a9c9fbb8', 18.50),
('Electric Toothbrush with 3 Heads', '3e97d950-b115-4a2f-a737-daa0a9c9fbb8', 39.99),
('Organic Lavender Essential Oil', '3e97d950-b115-4a2f-a737-daa0a9c9fbb8', 9.95),
('Digital Body Weight Scale', '3e97d950-b115-4a2f-a737-daa0a9c9fbb8', 25.00),

-- Automotive (ded87a8e-61af-4af6-91b0-3fe88eabde27)
('Digital Tire Pressure Gauge', 'ded87a8e-61af-4af6-91b0-3fe88eabde27', 14.99),
('Car Phone Mount Holder Universal', 'ded87a8e-61af-4af6-91b0-3fe88eabde27', 11.50),
('Microfiber Cleaning Cloths (12-Pack)', 'ded87a8e-61af-4af6-91b0-3fe88eabde27', 9.99),
('Portable Car Jump Starter Battery Pack', 'ded87a8e-61af-4af6-91b0-3fe88eabde27', 69.95),

-- Computers (2a04b7db-ed43-4345-9708-63bd70639db2)
('Wireless Keyboard and Mouse Combo', '2a04b7db-ed43-4345-9708-63bd70639db2', 34.99),
('External SSD Hard Drive 1TB USB 3.1', '2a04b7db-ed43-4345-9708-63bd70639db2', 99.50),
('Laptop Stand Adjustable Ergonomic', '2a04b7db-ed43-4345-9708-63bd70639db2', 28.00),
('Webcam 1080p with Microphone', '2a04b7db-ed43-4345-9708-63bd70639db2', 32.95),

-- Mobile Phones (d3ca4f5c-46c1-48d5-8fc8-42ad9145b476)
('Smartphone Model X 128GB (Unlocked)', 'd3ca4f5c-46c1-48d5-8fc8-42ad9145b476', 699.00),
('Screen Protector Tempered Glass (2-Pack)', 'd3ca4f5c-46c1-48d5-8fc8-42ad9145b476', 8.99),
('Power Bank 20000mAh Fast Charging', 'd3ca4f5c-46c1-48d5-8fc8-42ad9145b476', 35.50),
('Wireless Earbuds Bluetooth 5.2', 'd3ca4f5c-46c1-48d5-8fc8-42ad9145b476', 45.99),

-- Fashion Accessories (774bafa6-324f-4388-938e-d334c5408160)
('Leather Wallet RFID Blocking', '774bafa6-324f-4388-938e-d334c5408160', 29.95),
('Classic Aviator Sunglasses Polarized', '774bafa6-324f-4388-938e-d334c5408160', 19.99),
('Silk Scarf Floral Pattern', '774bafa6-324f-4388-938e-d334c5408160', 24.50),
('Stainless Steel Analog Watch', '774bafa6-324f-4388-938e-d334c5408160', 89.00),

-- Furniture (30e31326-510d-42f8-8da2-6f3cb5f7945a)
('Modern Office Desk Chair Ergonomic', '30e31326-510d-42f8-8da2-6f3cb5f7945a', 149.99),
('Bookshelf 5-Tier Industrial Style', '30e31326-510d-42f8-8da2-6f3cb5f7945a', 75.50),
('Coffee Table with Storage Shelf', '30e31326-510d-42f8-8da2-6f3cb5f7945a', 99.00),
('Memory Foam Mattress Topper Queen Size', '30e31326-510d-42f8-8da2-6f3cb5f7945a', 119.95),

-- Groceries (64045ffc-db29-4fc8-a277-4c8ac0f236d4)
('Organic Whole Bean Coffee 1kg', '64045ffc-db29-4fc8-a277-4c8ac0f236d4', 15.99),
('Extra Virgin Olive Oil 500ml', '64045ffc-db29-4fc8-a277-4c8ac0f236d4', 12.50),
('Almond Milk Unsweetened 1L', '64045ffc-db29-4fc8-a277-4c8ac0f236d4', 3.95),
('Gluten-Free Pasta 500g', '64045ffc-db29-4fc8-a277-4c8ac0f236d4', 4.25),

-- Music & Movies (db90f991-141e-401a-b5d9-01462759ff5f)
('Latest Blockbuster Movie Blu-ray', 'db90f991-141e-401a-b5d9-01462759ff5f', 24.99),
('Classic Rock Vinyl LP Record', 'db90f991-141e-401a-b5d9-01462759ff5f', 29.95),
('Streaming Service Gift Card $50', 'db90f991-141e-401a-b5d9-01462759ff5f', 50.00),
('Documentary Series DVD Box Set', 'db90f991-141e-401a-b5d9-01462759ff5f', 39.99),

-- Office Supplies (2841d04d-4c2e-41d7-9150-3d3e5f3914d4)
('Gel Pens Black Ink (12-Pack)', '2841d04d-4c2e-41d7-9150-3d3e5f3914d4', 10.99),
('Sticky Notes Assorted Colors (6-Pack)', '2841d04d-4c2e-41d7-9150-3d3e5f3914d4', 7.50),
('Heavy Duty Stapler', '2841d04d-4c2e-41d7-9150-3d3e5f3914d4', 18.95),
('Whiteboard Dry Erase Markers (8 Colors)', '2841d04d-4c2e-41d7-9150-3d3e5f3914d4', 9.99);

-- Add values into customers table
INSERT INTO lapsed_customer_problem.customers (name, gmail) VALUES
('Alice Johnson', 'alice.j.88@gmail.com'),
('Bob Williams', 'bob.williams.dev@gmail.com'),
('Catherine Brown', 'catherine.brown12@gmail.com'),
('David Jones', 'david.jones.ap@gmail.com'),
('Eleanor Garcia', 'eleanor.g@gmail.com'),
('Frank Miller', 'frank.miller.305@gmail.com'),
('Grace Davis', 'grace.davis.in@gmail.com'),
('Henry Rodriguez', 'h.rodriguez@gmail.com'),
('Isabella Martinez', 'isabella.martinez22@gmail.com'),
('Jack Hernandez', 'jack.hdz@gmail.com'),
('Katherine Lopez', 'k.lopez.work@gmail.com'),
('Liam Gonzalez', 'liam.gonzalez8@gmail.com'),
('Mia Wilson', 'mia.wilson.77@gmail.com'),
('Noah Anderson', 'noah.anderson.ap@gmail.com'),
('Olivia Thomas', 'olivia.t.19@gmail.com'),
('Peter Taylor', 'peter.taylor.in@gmail.com'),
('Quinn Moore', 'quinn.moore.q@gmail.com'),
('Ryan Martin', 'ryan.martin.2025@gmail.com'),
('Sophia Jackson', 'sophia.jackson.lee@gmail.com'),
('Thomas Lee', 'thomas.lee.01@gmail.com'),
('Uma Perez', 'uma.perez.u@gmail.com'),
('Victor Thompson', 'victor.thompson.v@gmail.com'),
('Willow White', 'willow.white.in@gmail.com'),
('Xavier Harris', 'xavier.harris.x@gmail.com'),
('Yara Sanchez', 'yara.sanchez.ap@gmail.com'),
('Zane Clark', 'zane.clark.z@gmail.com'),
('Amelia Ramirez', 'amelia.ramirez.ar@gmail.com'),
('Benjamin Lewis', 'benjamin.lewis.bl@gmail.com'),
('Chloe Robinson', 'chloe.robinson.cr@gmail.com'),
('Daniel Walker', 'daniel.walker.dw@gmail.com'),
('Emily Young', 'emily.young.ey@gmail.com'),
('Finn Allen', 'finn.allen.fa@gmail.com'),
('Gabriella King', 'gabriella.king.gk@gmail.com'),
('Harper Wright', 'harper.wright.hw@gmail.com'),
('Isaac Scott', 'isaac.scott.is@gmail.com'),
('Jade Green', 'jade.green.jg@gmail.com'),
('Kevin Adams', 'kevin.adams.ka@gmail.com'),
('Luna Baker', 'luna.baker.lb@gmail.com'),
('Mason Nelson', 'mason.nelson.mn@gmail.com'),
('Nora Carter', 'nora.carter.nc@gmail.com'),
('Owen Mitchell', 'owen.mitchell.om@gmail.com'),
('Penelope Perez', 'penelope.perez.pp@gmail.com'),
('Riley Roberts', 'riley.roberts.rr@gmail.com'),
('Samuel Turner', 'samuel.turner.st@gmail.com'),
('Tara Phillips', 'tara.phillips.tp@gmail.com'),
('Wyatt Campbell', 'wyatt.campbell.wc@gmail.com'),
('Victoria Parker', 'victoria.parker.vp@gmail.com'),
('Adam Evans', 'adam.evans.ae@gmail.com'),
('Bella Edwards', 'bella.edwards.be@gmail.com'),
('Caleb Collins', 'caleb.collins.cc@gmail.com'),
('Dylan Stewart', 'dylan.stewart.ds@gmail.com'),
('Evelyn Sanchez', 'evelyn.sanchez.es@gmail.com'),
('Grayson Morris', 'grayson.morris.gm@gmail.com'),
('Hazel Rogers', 'hazel.rogers.hr@gmail.com'),
('Julian Reed', 'julian.reed.jr@gmail.com'),
('Leah Cook', 'leah.cook.lc@gmail.com'),
('Miles Morgan', 'miles.morgan.mm@gmail.com'),
('Paisley Bell', 'paisley.bell.pb@gmail.com'),
('Sawyer Murphy', 'sawyer.murphy.sm@gmail.com'),
('Stella Bailey', 'stella.bailey.sb@gmail.com');

-- Add values into orders table
INSERT INTO lapsed_customer_problem.orders (customer_id, status) VALUES
('6ec370b1-20b7-4a1a-98d1-4a245036c5ea', 'completed'), -- Alice Johnson
('f56531ef-39ca-411a-85d0-0f78970cfca1', 'shipped'),   -- Bob Williams
('7d98933f-7752-4e76-bf71-1c1522663000', 'pending'),   -- David Jones
('b8b0c7ee-e110-442f-94ea-2702a48abeff', 'completed'), -- Frank Miller
('36853ec4-5c9d-4fbf-83ac-131542f0b333', 'ordered'),   -- Isabella Martinez
('d1549df2-b0f2-4277-ae67-94d6d7f138a3', 'shipped'),   -- Liam Gonzalez
('d4e54317-034b-4931-b9d6-8e21d94512c5', 'cancelled'), -- Olivia Thomas
('46f16f4e-b728-4572-8449-59a14d4549a3', 'completed'), -- Ryan Martin
('2e511bd2-d71d-41bf-bd75-9f086b5a469b', 'pending'),   -- Thomas Lee
('33d84525-581c-40f9-afe6-b0317044a95e', 'shipped'),   -- Willow White
('fa46b409-c1bb-4280-bbe1-ddf26c885047', 'completed'), -- Zane Clark
('4217f2b2-f4b6-44df-aaad-33fb56219928', 'pending'),   -- Chloe Robinson
('64c4bc15-fdfb-4a87-aa31-9d5469dd750e', 'shipped'),   -- Gabriella King
('93005021-d470-466e-8585-6e3c8c3f9b12', 'completed'), -- Kevin Adams
('0b707139-9769-4596-88b9-7174a700e974', 'ordered'),   -- Owen Mitchell
('6ec370b1-20b7-4a1a-98d1-4a245036c5ea', 'shipped'),   -- Alice Johnson (Order 2)
('51ec5aa6-ad92-4592-8228-eb8c7f7c7444', 'completed'), -- Peter Taylor
('e4a97659-d70d-49c5-9a2f-fcbcd803683b', 'pending'),   -- Sophia Jackson
('90297f97-857a-4adb-a955-cacb598ce03b', 'ordered'),   -- Victor Thompson
('63709839-2cca-487f-9fd2-351a88e5331b', 'completed'), -- Yara Sanchez
('f87c925e-d66b-4d0f-8eef-277052ff5f88', 'shipped'),   -- Benjamin Lewis
('3fb6f3be-5b51-4762-8266-a228d9644356', 'cancelled'), -- Daniel Walker
('1789085a-fda7-413a-a1c8-39cb2cb4e9ad', 'pending'),   -- Emily Young
('79e8bc09-4b78-49c1-a857-99c5e41fed6f', 'completed'), -- Harper Wright
('da9eb938-ad51-4fe9-95d3-0d7d3ec7604e', 'ordered'),   -- Jade Green
('7c943b2c-476e-4a23-bd67-8f91f30aca79', 'shipped'),   -- Luna Baker
('d8a6cb05-fc05-4711-88fc-8c1c0cfcd0db', 'completed'), -- Nora Carter
('43b0473a-6aea-45b6-92e4-28c49f85fd38', 'pending'),   -- Riley Roberts
('573cdff1-98ee-4365-971d-a2614d1bb3b7', 'shipped'),   -- Tara Phillips
('2e4a8208-5fca-4311-b91e-7bec910e4ba4', 'completed'), -- Victoria Parker
('14060b96-9ec3-415d-a0f0-93e6b08318e3', 'ordered'),   -- Caleb Collins
('df653aae-0186-4eae-bbb9-c6b37d3dbc89', 'shipped'),   -- Hazel Rogers
('9cfc8739-b29e-465e-ad27-76eb5570f33e', 'completed'), -- Miles Morgan
('dc42490c-a189-4054-910e-e352d5c3eb7c', 'pending'),   -- Stella Bailey
('f56531ef-39ca-411a-85d0-0f78970cfca1', 'completed'); -- Bob Williams (Order 2)

-- Add values into order_items table
INSERT INTO lapsed_customer_problem.order_items (order_id, product_id, quantity, total_product_price) VALUES
-- Order: 07b445c8-3ab9-4718-86ce-b7b94a9ba45c
('07b445c8-3ab9-4718-86ce-b7b94a9ba45c', 'ffb7168b-6c0f-4804-b00d-00e532666e68', 1, 199.99), -- Headphones (1 * 199.99)
('07b445c8-3ab9-4718-86ce-b7b94a9ba45c', '22c40bca-e652-440f-bc1c-b030d8486a09', 2, 25.98),  -- Cable (2 * 12.99)

-- Order: ca290a7f-0169-4b4e-99f8-e9cb92fb5ced
('ca290a7f-0169-4b4e-99f8-e9cb92fb5ced', 'be17f4d6-235f-43d1-9a4f-2b9d15fa6849', 1, 19.95),  -- T-Shirt (1 * 19.95)

-- Order: fde2fea9-785f-48bd-a427-7cfdf27dbf43
('fde2fea9-785f-48bd-a427-7cfdf27dbf43', 'f2803fbc-2915-420a-8f7f-cea83afce7a2', 1, 16.99),  -- Book (Atomic Habits) (1 * 16.99)
('fde2fea9-785f-48bd-a427-7cfdf27dbf43', '3f62f804-f196-481a-8fa6-c05e92c1e3d0', 1, 18.75),  -- Mugs (1 * 18.75)

-- Order: 35f32757-db13-4095-88ed-a63366565370
('35f32757-db13-4095-88ed-a63366565370', 'b35f45aa-3b46-428e-9ad2-28e40e49c8d4', 1, 89.99),  -- Drill (1 * 89.99)

-- Order: 80e1b2f1-dd70-44b7-8552-fbf44c114faa
('80e1b2f1-dd70-44b7-8552-fbf44c114faa', 'af1c494c-ed3b-4e24-a3a7-52757bed8c50', 1, 24.99),  -- Yoga Mat (1 * 24.99)

-- Order: 7eb124c5-170d-41da-8c95-dc762bdd6a1a
('7eb124c5-170d-41da-8c95-dc762bdd6a1a', 'e408e308-ad61-42b4-95d7-e8bba2e030bd', 1, 44.99),  -- Jeans (1 * 44.99)
('7eb124c5-170d-41da-8c95-dc762bdd6a1a', 'be17f4d6-235f-43d1-9a4f-2b9d15fa6849', 2, 39.90),  -- T-Shirt (2 * 19.95)

-- Order: 6581bd7c-91d9-411d-845d-52d27e21e023 (Cancelled)
('6581bd7c-91d9-411d-845d-52d27e21e023', '1dacd57c-2322-46c8-9ca5-67c0a563ed94', 1, 18.50),  -- VitC Serum (1 * 18.50)

-- Order: 057b5fb1-bb3b-4c53-8328-466ffdcf894f
('057b5fb1-bb3b-4c53-8328-466ffdcf894f', 'e73859b5-3b9e-4d6f-a1dd-9b785c62b086', 1, 14.99),  -- Tire Gauge (1 * 14.99)
('057b5fb1-bb3b-4c53-8328-466ffdcf894f', 'c3215017-8acc-45d3-837f-ac5b0bc5b4a0', 1, 29.95),  -- Wallet (1 * 29.95)

-- Order: eefcdc4d-65dc-45d2-8bfa-f335fd41b06f
('eefcdc4d-65dc-45d2-8bfa-f335fd41b06f', 'fa0be483-453a-48e3-a436-5a5fed1f0836', 1, 34.99),  -- KB+Mouse (1 * 34.99)

-- Order: 79c28ec7-935b-43ff-ae69-a93e08f41fb6
('79c28ec7-935b-43ff-ae69-a93e08f41fb6', '29167f81-d62e-48ac-8454-4cd02300ff7a', 1, 32.95),  -- Webcam (1 * 32.95)
('79c28ec7-935b-43ff-ae69-a93e08f41fb6', 'f87cb270-7f05-40ff-90a1-146df0cdd9cf', 3, 32.97),  -- Gel Pens (3 * 10.99)

-- Order: ae9d765f-a661-4b0d-85ee-66bd3f465a43
('ae9d765f-a661-4b0d-85ee-66bd3f465a43', 'b726727f-5e11-4606-8ef2-f8ca43bf843e', 1, 149.99), -- Office Chair (1 * 149.99)

-- Order: fdd20139-d1b2-4969-8fd0-5b6960e49a38
('fdd20139-d1b2-4969-8fd0-5b6960e49a38', '2e4b1b26-8ecf-4974-806a-37d7ad6703c0', 1, 35.50),  -- Power Bank (1 * 35.50)

-- Order: 0b90a52e-c3c0-4f40-81ea-21b0c6f2bf23
('0b90a52e-c3c0-4f40-81ea-21b0c6f2bf23', '8332eb40-94c3-40f3-a9bb-884ab87957f7', 2, 31.98),  -- Coffee Beans (2 * 15.99)
('0b90a52e-c3c0-4f40-81ea-21b0c6f2bf23', '3f62f804-f196-481a-8fa6-c05e92c1e3d0', 1, 18.75),  -- Mugs (1 * 18.75)

-- Order: 1bb89555-35b4-4b5c-9569-54ee57f66263
('1bb89555-35b4-4b5c-9569-54ee57f66263', 'deabc82b-43b6-4818-bafa-84d18ddd3b9e', 1, 24.99),  -- Blu-ray (1 * 24.99)
('1bb89555-35b4-4b5c-9569-54ee57f66263', 'f2803fbc-2915-420a-8f7f-cea83afce7a2', 1, 16.99),  -- Book (Atomic Habits) (1 * 16.99)

-- Order: 84c47e0c-7777-42eb-b50b-f7bd185189c6
('84c47e0c-7777-42eb-b50b-f7bd185189c6', 'af1c494c-ed3b-4e24-a3a7-52757bed8c50', 1, 24.99),  -- Yoga Mat (1 * 24.99)
('84c47e0c-7777-42eb-b50b-f7bd185189c6', '22c40bca-e652-440f-bc1c-b030d8486a09', 1, 12.99),  -- Cable (1 * 12.99)

-- Order: 536db879-836c-4f0e-8038-80c91d68906b
('536db879-836c-4f0e-8038-80c91d68906b', 'e408e308-ad61-42b4-95d7-e8bba2e030bd', 1, 44.99),  -- Jeans (1 * 44.99)

-- Order: 3152a710-61a2-4fdf-9511-3ce5172aa54f
('3152a710-61a2-4fdf-9511-3ce5172aa54f', 'fa0be483-453a-48e3-a436-5a5fed1f0836', 1, 34.99),  -- KB+Mouse (1 * 34.99)

-- Order: 56edd1d8-9571-46c6-990b-cd2ba73c95b0
('56edd1d8-9571-46c6-990b-cd2ba73c95b0', '1dacd57c-2322-46c8-9ca5-67c0a563ed94', 2, 37.00),  -- VitC Serum (2 * 18.50)

-- Order: 006b0605-cda4-40e9-a881-c59617ac5199
('006b0605-cda4-40e9-a881-c59617ac5199', 'c3215017-8acc-45d3-837f-ac5b0bc5b4a0', 1, 29.95),  -- Wallet (1 * 29.95)

-- Order: ac8c11a5-21f3-4580-8138-f49715119891
('ac8c11a5-21f3-4580-8138-f49715119891', '29167f81-d62e-48ac-8454-4cd02300ff7a', 1, 32.95),  -- Webcam (1 * 32.95)

-- Order: 64a801fb-c46c-4916-8b99-04d269be9f75
('64a801fb-c46c-4916-8b99-04d269be9f75', 'b35f45aa-3b46-428e-9ad2-28e40e49c8d4', 1, 89.99),  -- Drill (1 * 89.99)

-- Order: 4c58c30d-fa9b-443e-9f8b-8ccb35a91868 (Cancelled)
('4c58c30d-fa9b-443e-9f8b-8ccb35a91868', '8332eb40-94c3-40f3-a9bb-884ab87957f7', 1, 15.99),  -- Coffee Beans (1 * 15.99)

-- Order: cc7443fe-42f8-4991-91e6-841dd984007f
('cc7443fe-42f8-4991-91e6-841dd984007f', 'deabc82b-43b6-4818-bafa-84d18ddd3b9e', 1, 24.99),  -- Blu-ray (1 * 24.99)

-- Order: ac4748c2-cb1e-43cb-ad89-6e608aa26e77
('ac4748c2-cb1e-43cb-ad89-6e608aa26e77', 'f87cb270-7f05-40ff-90a1-146df0cdd9cf', 2, 21.98),  -- Gel Pens (2 * 10.99)
('ac4748c2-cb1e-43cb-ad89-6e608aa26e77', '3f62f804-f196-481a-8fa6-c05e92c1e3d0', 1, 18.75),  -- Mugs (1 * 18.75)

-- Order: d22f7cc9-c4dc-4a20-81e5-0eef95e8169b
('d22f7cc9-c4dc-4a20-81e5-0eef95e8169b', 'ffb7168b-6c0f-4804-b00d-00e532666e68', 1, 199.99), -- Headphones (1 * 199.99)

-- Order: 4fb48afc-bc8d-45d6-b013-b737a915120e
('4fb48afc-bc8d-45d6-b013-b737a915120e', 'be17f4d6-235f-43d1-9a4f-2b9d15fa6849', 3, 59.85),  -- T-Shirt (3 * 19.95)

-- Order: 73234b72-5e0b-46e9-8d4a-8cbf9c5b4217
('73234b72-5e0b-46e9-8d4a-8cbf9c5b4217', '22c40bca-e652-440f-bc1c-b030d8486a09', 1, 12.99),  -- Cable (1 * 12.99)

-- Order: b0942e2a-1b28-4e8d-8722-22013582f7a8
('b0942e2a-1b28-4e8d-8722-22013582f7a8', 'f2803fbc-2915-420a-8f7f-cea83afce7a2', 1, 16.99),  -- Book (Atomic Habits) (1 * 16.99)

-- Order: 39b6bf78-6cb3-4087-9a03-c5ef3b0a983b
('39b6bf78-6cb3-4087-9a03-c5ef3b0a983b', '2e4b1b26-8ecf-4974-806a-37d7ad6703c0', 1, 35.50),  -- Power Bank (1 * 35.50)

-- Order: 30feceb3-ff57-44c1-a6a3-7f061f0928f1
('30feceb3-ff57-44c1-a6a3-7f061f0928f1', 'b726727f-5e11-4606-8ef2-f8ca43bf843e', 1, 149.99), -- Office Chair (1 * 149.99)

-- Order: f174e8bd-5682-4e54-8285-d9aed41701c3
('f174e8bd-5682-4e54-8285-d9aed41701c3', 'af1c494c-ed3b-4e24-a3a7-52757bed8c50', 1, 24.99),  -- Yoga Mat (1 * 24.99)

-- Order: 08d7691d-7553-4c72-99c3-a8f9a19a239f
('08d7691d-7553-4c72-99c3-a8f9a19a239f', '1dacd57c-2322-46c8-9ca5-67c0a563ed94', 1, 18.50),  -- VitC Serum (1 * 18.50)

-- Order: 54c2b8f6-67bb-427e-bef8-8906201a6f72
('54c2b8f6-67bb-427e-bef8-8906201a6f72', 'e73859b5-3b9e-4d6f-a1dd-9b785c62b086', 1, 14.99),  -- Tire Gauge (1 * 14.99)

-- Order: 6315c9b8-8a3e-4789-9780-5ecf30582fa7
('6315c9b8-8a3e-4789-9780-5ecf30582fa7', 'c3215017-8acc-45d3-837f-ac5b0bc5b4a0', 1, 29.95),  -- Wallet (1 * 29.95)

-- Order: 34d9420f-8a49-432a-aefe-e27e9f762d90
('34d9420f-8a49-432a-aefe-e27e9f762d90', 'fa0be483-453a-48e3-a436-5a5fed1f0836', 1, 34.99),  -- KB+Mouse (1 * 34.99)
('34d9420f-8a49-432a-aefe-e27e9f762d90', '29167f81-d62e-48ac-8454-4cd02300ff7a', 1, 32.95);  -- Webcam (1 * 32.95)


-- Update created_at and updated_at for orders and order_items to resemble the 

-- 1. UPDATE the 'orders' table
UPDATE lapsed_customer_problem.orders
SET
    -- Generate timestamp WITHIN THE LAST 6 MONTHS
    created_at = to_timestamp(1696953840 + (random() * 15807000))::timestamptz::timestamp, -- Explicit cast
    -- Set updated_at similarly for consistency in this update
    updated_at = to_timestamp(1696953840 + (random() * 15807000))::timestamptz::timestamp  -- Explicit cast
WHERE id IN (
    -- Paste the list of your 35 order_ids here
    '07b445c8-3ab9-4718-86ce-b7b94a9ba45c', 'ca290a7f-0169-4b4e-99f8-e9cb92fb5ced',
    'fde2fea9-785f-48bd-a427-7cfdf27dbf43', '35f32757-db13-4095-88ed-a63366565370',
    '80e1b2f1-dd70-44b7-8552-fbf44c114faa', '7eb124c5-170d-41da-8c95-dc762bdd6a1a',
    '6581bd7c-91d9-411d-845d-52d27e21e023', '057b5fb1-bb3b-4c53-8328-466ffdcf894f',
    'eefcdc4d-65dc-45d2-8bfa-f335fd41b06f', '79c28ec7-935b-43ff-ae69-a93e08f41fb6',
    'ae9d765f-a661-4b0d-85ee-66bd3f465a43', 'fdd20139-d1b2-4969-8fd0-5b6960e49a38',
    '0b90a52e-c3c0-4f40-81ea-21b0c6f2bf23', '1bb89555-35b4-4b5c-9569-54ee57f66263',
    '84c47e0c-7777-42eb-b50b-f7bd185189c6', '536db879-836c-4f0e-8038-80c91d68906b',
    '3152a710-61a2-4fdf-9511-3ce5172aa54f', '56edd1d8-9571-46c6-990b-cd2ba73c95b0',
    '006b0605-cda4-40e9-a881-c59617ac5199', 'ac8c11a5-21f3-4580-8138-f49715119891',
    '64a801fb-c46c-4916-8b99-04d269be9f75', '4c58c30d-fa9b-443e-9f8b-8ccb35a91868',
    'cc7443fe-42f8-4991-91e6-841dd984007f', 'ac4748c2-cb1e-43cb-ad89-6e608aa26e77',
    'd22f7cc9-c4dc-4a20-81e5-0eef95e8169b', '4fb48afc-bc8d-45d6-b013-b737a915120e',
    '73234b72-5e0b-46e9-8d4a-8cbf9c5b4217', 'b0942e2a-1b28-4e8d-8722-22013582f7a8',
    '39b6bf78-6cb3-4087-9a03-c5ef3b0a983b', '30feceb3-ff57-44c1-a6a3-7f061f0928f1',
    'f174e8bd-5682-4e54-8285-d9aed41701c3', '08d7691d-7553-4c72-99c3-a8f9a19a239f',
    '54c2b8f6-67bb-427e-bef8-8906201a6f72', '6315c9b8-8a3e-4789-9780-5ecf30582fa7',
    '34d9420f-8a49-432a-aefe-e27e9f762d90'
);

-- 2. UPDATE the 'order_items' table
UPDATE lapsed_customer_problem.order_items
SET
    -- Generate timestamp WITHIN THE LAST 6 MONTHS
    created_at = to_timestamp(1696953840 + (random() * 15807000))::timestamptz::timestamp, -- Explicit cast
    -- Set updated_at similarly for consistency in this update
    updated_at = to_timestamp(1696953840 + (random() * 15807000))::timestamptz::timestamp  -- Explicit cast
WHERE order_id IN (
    -- Paste the same list of your 35 order_ids here
    '07b445c8-3ab9-4718-86ce-b7b94a9ba45c', 'ca290a7f-0169-4b4e-99f8-e9cb92fb5ced',
    'fde2fea9-785f-48bd-a427-7cfdf27dbf43', '35f32757-db13-4095-88ed-a63366565370',
    '80e1b2f1-dd70-44b7-8552-fbf44c114faa', '7eb124c5-170d-41da-8c95-dc762bdd6a1a',
    '6581bd7c-91d9-411d-845d-52d27e21e023', '057b5fb1-bb3b-4c53-8328-466ffdcf894f',
    'eefcdc4d-65dc-45d2-8bfa-f335fd41b06f', '79c28ec7-935b-43ff-ae69-a93e08f41fb6',
    'ae9d765f-a661-4b0d-85ee-66bd3f465a43', 'fdd20139-d1b2-4969-8fd0-5b6960e49a38',
    '0b90a52e-c3c0-4f40-81ea-21b0c6f2bf23', '1bb89555-35b4-4b5c-9569-54ee57f66263',
    '84c47e0c-7777-42eb-b50b-f7bd185189c6', '536db879-836c-4f0e-8038-80c91d68906b',
    '3152a710-61a2-4fdf-9511-3ce5172aa54f', '56edd1d8-9571-46c6-990b-cd2ba73c95b0',
    '006b0605-cda4-40e9-a881-c59617ac5199', 'ac8c11a5-21f3-4580-8138-f49715119891',
    '64a801fb-c46c-4916-8b99-04d269be9f75', '4c58c30d-fa9b-443e-9f8b-8ccb35a91868',
    'cc7443fe-42f8-4991-91e6-841dd984007f', 'ac4748c2-cb1e-43cb-ad89-6e608aa26e77',
    'd22f7cc9-c4dc-4a20-81e5-0eef95e8169b', '4fb48afc-bc8d-45d6-b013-b737a915120e',
    '73234b72-5e0b-46e9-8d4a-8cbf9c5b4217', 'b0942e2a-1b28-4e8d-8722-22013582f7a8',
    '39b6bf78-6cb3-4087-9a03-c5ef3b0a983b', '30feceb3-ff57-44c1-a6a3-7f061f0928f1',
    'f174e8bd-5682-4e54-8285-d9aed41701c3', '08d7691d-7553-4c72-99c3-a8f9a19a239f',
    '54c2b8f6-67bb-427e-bef8-8906201a6f72', '6315c9b8-8a3e-4789-9780-5ecf30582fa7',
    '34d9420f-8a49-432a-aefe-e27e9f762d90'
);

```

3. Solution for the problem

```sql
WITH OrdersWithCompleted AS
(
	SELECT o.id, o.customer_id as customer_id, cus.name as name, cus.gmail as gmail, SUM(oi.total_product_price) as order_sum, o.created_at as created_at 
	FROM lapsed_customer_problem.order_items oi
	JOIN lapsed_customer_problem.orders o ON oi.order_id = o.id
	JOIN lapsed_customer_problem.customers cus ON o.customer_id = cus.id
	WHERE
	o.status = 'completed'
	GROUP BY o.id, cus.name, cus.gmail
),
CompletedOrderWithTotalSumOrder AS
(
	SELECT SUM(order_sum) as total_sum, dat.customer_id, dat.name, dat.gmail, MAX(dat.created_at) as last_purchased_date from
	OrdersWithCompleted dat
	GROUP BY dat.customer_id, dat.name, dat.gmail
	ORDER BY total_sum DESC
),
CountOfTotal AS
(
	SELECT count(*) as user_count FROM CompletedOrderWithTotalSumOrder
),
CompletedOrderWithTotalSumOrderWithLimit AS
(
	SELECT dex.customer_id, dex.name, dex.gmail, dex.total_sum, dex.last_purchased_date
	FROM CompletedOrderWithTotalSumOrder dex
	LIMIT CAST(CEIL((SELECT user_count FROM CountOfTotal)/10)AS INTEGER)
)
SELECT dex.customer_id, dex.name, dex.gmail, dex.total_sum, dex.last_purchased_date
FROM CompletedOrderWithTotalSumOrderWithLimit dex
WHERE dex.last_purchased_date <= NOW() - INTERVAL '6 months'
```
