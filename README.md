# coding-challenge-mysql

## Question 1: Total Sales Revenue by Product

SELECT p.id AS product_id, 
       p.name AS product_name, 
       SUM(oi.quantity * oi.price) AS total_revenue
FROM products p
JOIN order_items oi ON p.id = oi.product_id
GROUP BY p.id
ORDER BY total_revenue DESC;

## Question 2: Top Customers by Spending

SELECT c.id AS customer_id, 
       c.name AS customer_name, 
       SUM(oi.quantity * oi.price) AS total_spending
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN order_items oi ON o.id = oi.order_id
GROUP BY c.id
ORDER BY total_spending DESC
LIMIT 5;

## Question 3: Average Order Value per Customer

SELECT c.id AS customer_id, 
       c.name AS customer_name, 
       AVG(o.total_amount) AS average_order_value
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id
HAVING COUNT(o.id) > 0
ORDER BY average_order_value DESC;

## Question 4: Recent Orders

SELECT o.id AS order_id, 
       c.name AS customer_name, 
       o.order_date, 
       o.status AS order_status
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.order_date >= NOW() - INTERVAL 30 DAY
ORDER BY o.order_date DESC;


## Question 5: Running Total of Customer Spending

WITH RunningTotal AS (
    SELECT o.customer_id, 
           o.id AS order_id, 
           o.order_date, 
           SUM(oi.quantity * oi.price) AS order_total, 
           SUM(SUM(oi.quantity * oi.price)) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS running_total
    FROM orders o
    JOIN order_items oi ON o.id = oi.order_id
    GROUP BY o.customer_id, o.id
)
SELECT customer_id, order_id, order_date, order_total, running_total
FROM RunningTotal
ORDER BY customer_id, order_date;


## Question 6: Product Review Summary

SELECT p.id AS product_id, 
       p.name AS product_name, 
       AVG(r.rating) AS average_rating, 
       COUNT(r.id) AS total_reviews
FROM products p
LEFT JOIN reviews r ON p.id = r.product_id
GROUP BY p.id
ORDER BY average_rating DESC, total_reviews DESC;


## Question 7: Customers Without Orders

SELECT c.id AS customer_id, 
       c.name AS customer_name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;


## Question 8: Update Last Purchased Date

UPDATE products p
JOIN order_items oi ON p.id = oi.product_id
JOIN orders o ON oi.order_id = o.id
SET p.last_purchased = o.order_date
WHERE o.order_date = (SELECT MAX(order_date) FROM orders WHERE id IN (SELECT order_id FROM order_items WHERE product_id = p.id));


## Question 9: Transaction Scenario

START TRANSACTION;

-- Step 1: Deduct the ordered quantity from a product’s stock
UPDATE products p
SET p.stock = p.stock - ? -- Ordered quantity
WHERE p.id = ?; -- Product ID

-- Step 2: Insert a new record in the orders table
INSERT INTO orders (customer_id, order_date, status)
VALUES (?, NOW(), 'Pending'); -- Customer ID and Order Status

-- Step 3: Insert one or more records in the order_items table
INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES (LAST_INSERT_ID(), ?, ?, ?); -- Order ID, Product ID, Quantity, Price

-- Step 4: Update the product’s last_purchased timestamp
UPDATE products p
SET p.last_purchased = NOW()
WHERE p.id = ?;

-- Error Handling: If any of the queries fail, rollback the transaction
COMMIT;
-- If any error occurs, rollback:
-- ROLLBACK;


## Question 10: Query Optimization and Indexing (Short Answer)
Using EXPLAIN Statement:

To analyze the performance of a query, we can use the EXPLAIN statement to get details about how MySQL executes the query.


EXPLAIN SELECT p.id AS product_id, 
               p.name AS product_name, 
               SUM(oi.quantity * oi.price) AS total_revenue
FROM products p
JOIN order_items oi ON p.id = oi.product_id
GROUP BY p.id
ORDER BY total_revenue DESC;


## Question 11: Query Optimization Challenge

SELECT 
    c.id AS customer_id, 
    c.name,
    (SELECT SUM(oi.quantity * oi.price)
     FROM order_items oi
     WHERE oi.order_id IN (
         SELECT o.id
         FROM orders o
         WHERE o.customer_id = c.id
     )
    ) AS total_spent
FROM customers c
WHERE c.id IN (SELECT DISTINCT customer_id FROM orders)
ORDER BY total_spent DESC;