## Table Of Contents  
[ERD](#erd)  
[Pizza Metrics](#a-pizza-metrics)  
[Runner & Customer](#b-runner-and-customer-experience)

### ERD  
![](Week2/ERD.png)

## _REQUIREMENTS_  
## A. Pizza Metrics  
- How many pizzas were ordered?  
- How many unique customer orders were made?  
- How many successful orders were delivered by each runner?  
- How many of each type of pizza was delivered?  
- How many Vegetarian and Meatlovers were ordered by each customer?  
- What was the maximum number of pizzas delivered in a single order?  
- For each customer, how many delivered pizzas had at least 1 change and how many had no changes?  
- How many pizzas were delivered that had both exclusions and extras?  
- What was the total volume of pizzas ordered for each hour of the day?  
- What was the volume of orders for each day of the week?

## Queries
```
--   A. Pizza Metrics
-- How many pizzas were ordered?
SELECT COUNT(pizza_id) AS total_pizza
FROM customer_orders;

-- How many unique customer orders were made?
SELECT COUNT(DISTINCT order_id) AS unique_order
FROM customer_orders;

-- How many successful orders were delivered by each runner?
SELECT runner_id , COUNT( run.order_id) AS successfull
FROM runner_orders run 
WHERE pickup_time <> 'null'
GROUP BY runner_id;

-- How many of each type of pizza was delivered?
WITH orders AS (
	SELECT order_id
	FROM runner_orders run 
	WHERE pickup_time <> 'null'
), piz_id AS (
	SELECT pizza_id, COUNT(*) AS pizzas
	FROM orders JOIN customer_orders cus ON orders.order_id = cus.order_id
	GROUP BY pizza_id
)	SELECT pizza_name, pizzas
	FROM piz_id JOIN pizza_names p_name ON piz_id.pizza_id = p_name.pizza_id

-- How many Vegetarian and Meatlovers were ordered by each customer?
SELECT customer_id, pizza_name,COUNT(*) AS total_pizzas
FROM pizza_names piz JOIN customer_orders cus ON piz.pizza_id = cus.pizza_id
WHERE pizza_name = 'Vegetarian' OR pizza_name = 'Meatlovers'
GROUP BY customer_id,pizza_name;

-- What was the maximum number of pizzas delivered in a single order?
SELECT run.order_id, COUNT(*) AS total_pizzas
FROM runner_orders run 
	JOIN customer_orders cus ON run.order_id = cus.order_id
WHERE pickup_time <> 'null'
GROUP BY run.order_id
ORDER BY COUNT(*) DESC
LIMIT 1

-- For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
SELECT cus.customer_id, 
SUM(CASE WHEN (
	(exclusions IS NOT NULL AND exclusions <> 'null' AND LENGTH(exclusions) > 0) 
	OR 
	(extras IS NOT NULL AND extras <> 'null' AND LENGTH(extras) > 0)) = true THEN 1 ELSE 0 END) AS changes,
SUM(CASE WHEN (
	(exclusions IS NOT NULL AND exclusions <> 'null' AND LENGTH(exclusions) > 0) 
	OR 
	(extras IS NOT NULL AND extras <> 'null' AND LENGTH(extras) > 0)) = true THEN 0 ELSE 1 END) AS no_changes
FROM runner_orders run 
	JOIN customer_orders cus ON run.order_id = cus.order_id
WHERE pickup_time <> 'null'
GROUP BY cus.customer_id

-- How many pizzas were delivered that had both exclusions and extras?
SELECT pizza_id, SUM(CASE WHEN ((exclusions IS NOT NULL AND exclusions <> 'null' AND LENGTH(exclusions) > 0) 
	AND
	(extras IS NOT NULL AND extras <> 'null' AND LENGTH(extras) > 0)) = true THEN 1 ELSE 0 END) AS both_changed
FROM runner_orders run 
	JOIN customer_orders cus ON run.order_id = cus.order_id
WHERE pickup_time <> 'null'
GROUP BY pizza_id;

-- What was the total volume of pizzas ordered for each hour of the day?
SELECT EXTRACT(HOUR FROM order_time) AS hour, COUNT(*) AS total_pizzas
FROM customer_orders 
GROUP BY 1;

-- What was the volume of orders for each day of the week? 
SELECT DATE_PART('dow',order_time) AS day, to_char(order_time, 'Dy') AS day_name, COUNT(*) AS total_orders
FROM customer_orders 
GROUP BY 1,2;
```  

## B. Runner and Customer Experience  
- How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)  
- What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
- Is there any relationship between the number of pizzas and how long the order takes to prepare?
- What was the average distance travelled for each customer?
- What was the difference between the longest and shortest delivery times for all orders?
- What was the average speed for each runner for each delivery and do you notice any trend for these values?
- What is the successful delivery percentage for each runner?

```
--   B. Runner and Customer Experience
-- How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
SELECT DATE_PART('week', registration_date + 3) AS week, COUNT(*) AS runners
FROM runners
GROUP BY 1

-- What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
WITH order_time_diff AS (
 SELECT DISTINCT cus.order_id, (run.pickup_time::timestamp - cus.order_time) AS time_diff 
 FROM runner_orders run JOIN customer_orders cus ON run.order_id = cus.order_id
 WHERE pickup_time <> 'null'
 )
 SELECT DATE_TRUNC('minute', AVG(time_diff) + '30 second') AS avg_time
 FROM order_time_diff;


-- Is there any relationship between the number of pizzas and how long the order takes to prepare?
WITH orders_group AS (
   SELECT cus.order_id,COUNT(cus.order_id) AS pizza_count ,(run.pickup_time::timestamp - cus.order_time) AS time_diff 
 FROM runner_orders run JOIN customer_orders cus ON run.order_id = cus.order_id
 WHERE pickup_time <> 'null'
 GROUP BY cus.order_id, run.pickup_time, cus.order_time
 )
 SELECT pizza_count, AVG(time_diff)
 FROM orders_group
 GROUP BY pizza_count
 
-- What was the average distance travelled for each customer?
SELECT customer_id, ROUND(AVG(TRIM('km' FROM distance)::numeric),2) AS avg_distance
FROM runner_orders run JOIN customer_orders cus ON run.order_id = cus.order_id
WHERE distance <> 'null'
GROUP BY customer_id;

-- What was the difference between the longest and shortest delivery times for all orders?
SELECT MAX(edit_duration) - MIN(edit_duration) AS delivery_time_diff
FROM (
	SELECT *, LEFT(split_part(duration, ' ',1),2)::numeric AS edit_duration
	FROM runner_orders
	WHERE duration <> 'null'
) sub;

-- What was the average speed for each runner for each delivery and do you notice any trend for these values?
SELECT runner_id, order_id, ROUND(AVG(km/hour),2) AS avg_speed
FROM (
	SELECT *, TRIM('km' FROM distance)::numeric AS km, LEFT(split_part(duration, ' ',1),2)::numeric / 60 AS hour
	FROM runner_orders
	WHERE pickup_time <> 'null'
) sub
GROUP BY runner_id, order_id
ORDER BY runner_id, order_id;
-->>> Of concern is Runner 2’s speed. There is a large variance between the lowest(35.1km/hr) and highest speeds (93.6km/hr).

-- What is the successful delivery percentage for each runner?
SELECT runner_id, ROUND(SUM(CASE WHEN distance <> 'null' THEN 1 ELSE 0 END)::numeric / COUNT(runner_id) * 100,2) AS delivery_percentage
FROM runner_orders
GROUP BY runner_id
;
```
