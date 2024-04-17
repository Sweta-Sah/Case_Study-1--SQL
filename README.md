# Danny's Dinner

Situation:
Danny's Diner, specializing in sushi, curry, and ramen, seeks assistance in leveraging basic customer data to enhance business operations. Danny aims to understand customer visiting patterns, expenditure, and favorite menu items to personalize the dining experience. Insights will guide decisions on expanding the loyalty program.

TASK:
Danny has shared three crucial datasets for our case study: sales, menu, and members. Due to privacy concerns, he provided samples of overall customer data. We aim to craft fully functioning SQL queries using these examples to address Danny's questions and improve his restaurant's operations.

ACTION:

<img width="449" alt="image" src="https://github.com/Sweta-Sah/Case_Study-1--SQL/assets/132820867/f0a34e75-f759-47c5-a3d8-c5681d7c481e">

CREATE SCHEMA dannys_diner;
SET search_path = dannys_diner;

CREATE TABLE sales (
  "customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_id" INTEGER
);

INSERT INTO sales
  ("customer_id", "order_date", "product_id")
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  "product_id" INTEGER,
  "product_name" VARCHAR(5),
  "price" INTEGER
);

INSERT INTO menu
  ("product_id", "product_name", "price")
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');

Result:
-- What is the total amount each customer spent at the restaurant?
SELECT s.customer_id, SUM(m.price)
FROM sales s
JOIN menu m
ON s.product_id=m.product_id
GROUP BY s.customer_id;

<img width="138" alt="image" src="https://github.com/Sweta-Sah/Case_Study-1--SQL/assets/132820867/1b836d61-30d0-4566-8c70-3ff72c6c2f2e">

-- How many days has each customer visited the restaurant?
SELECT customer_id, COUNT(DISTINCT order_date) AS no_of_visit
FROM sales
GROUP BY customer_id;

-- What was the first item from the menu purchased by each customer?
WITH cte AS (SELECT s.customer_id, s.order_date, m.product_name,
dense_rank() OVER(PARTITION BY customer_id ORDER BY order_date) AS r
FROM sales s
JOIN menu m
ON s.product_id=m.product_id
Group by s.customer_id, m.product_name, s.order_date)

SELECT cte.customer_id, cte.product_name
FROM cte
WHERE cte.r = 1;

-- What is the most purchased item on the menu and how many times was it purchased by all customers?
SELECT m.product_name, COUNT(s.product_id) AS purchase_count
FROM sales s
JOIN menu m
ON s.product_id=m.product_id
GROUP BY m.product_name
ORDER BY purchase_count DESC
LIMIT 1;

-- Which item was the most popular for each customer?
WITH cte AS (SELECT s.customer_id, m.product_name, COUNT(s.product_id) AS purchase_count,
dense_rank() OVER(PARTITION BY customer_id ORDER BY(SELECT COUNT(s.product_id) AS purchase_count)) AS r
FROM sales s
JOIN menu m
ON s.product_id=m.product_id
GROUP BY s.customer_id, m.product_name)

SELECT cte.customer_id, cte.product_name
FROM cte
WHERE r=1;

-- Which item was purchased first by the customer after they became a member?
WITH cte AS(SELECT DISTINCT s.customer_id, s.product_id, s.order_date,
dense_rank() OVER(PARTITION BY customer_id ORDER BY order_date) AS r
FROM sales s
JOIN members m
ON s.order_date>=m.join_date
WHERE s.customer_id IN (SELECT customer_id FROM members)),

cte2 AS(SELECT product_id, product_name FROM menu)

SELECT cte.customer_id, cte2.product_name, cte.order_date
FROM cte
JOIN cte2 ON cte.product_id=cte2.product_id
WHERE r=1
ORDER BY cte.customer_id;

-- Which item was purchased just before the customer became a member?
WITH cte AS(SELECT s.customer_id, s.product_id, s.order_date,
dense_rank() OVER(PARTITION BY customer_id ORDER BY order_date DESC) AS r
FROM sales s
JOIN members m
ON s.customer_id=m.customer_id
WHERE s.customer_id IN (SELECT customer_id FROM members) AND s.order_date < m.join_date),

cte2 AS(SELECT product_id, product_name FROM menu)

SELECT cte.customer_id, cte2.product_name, cte.order_date
FROM cte
JOIN cte2 ON cte.product_id=cte2.product_id
WHERE r=1
ORDER BY cte.customer_id;

-- What is the total items and amount spent for each member before they became a member?
SELECT s.customer_id, COUNT(s.product_id) AS total_item, SUM(m.price) AS total_amount
FROM sales s
JOIN menu m
ON s.product_id=m.product_id
JOIN members mem
ON s.customer_id=mem.customer_id
WHERE s.customer_id IN (SELECT customer_id FROM members) AND s.order_date < mem.join_date
GROUP BY s.customer_id
ORDER BY s.customer_id;

-- If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
WITH cte AS(SELECT s.customer_id, s.product_id, m.price,
CASE when s.product_id=1 THEN m.price*10*2 ELSE m.price*10 END AS points 
FROM sales s
JOIN menu m
ON s.product_id=m.product_id)

SELECT cte.customer_id, SUM(cte.points) AS total_points
FROM cte
GROUP BY cte.customer_id;

-- In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
WITH cte AS(SELECT s.customer_id, s.product_id, m.price, s.order_date,
CASE when s.product_id=1 THEN m.price*10*2 
WHEN s.order_date BETWEEN mem.join_date AND DATE_ADD(mem.join_date, INTERVAL 6 DAY) THEN m.price*10*2 
ELSE m.price*10 END AS points 
FROM sales s
JOIN menu m
ON s.product_id=m.product_id
JOIN members mem
ON s.customer_id=mem.customer_id
WHERE MONTH(order_date)= 1)

SELECT cte.customer_id, SUM(cte.points) AS total_points
FROM cte
GROUP BY cte.customer_id
ORDER BY customer_id;
