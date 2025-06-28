# EDA-Coffee-Expansion-using-SQL

## Table of contents

- [Project Overview](#project-overview)
- [Data Sources](#data-sources)
- [Tools](#tools)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [Data Analysis](#data-analysis)
- [Findings](#findings)
- [Recommendations](#recommendations)
- [References](#references)

### Project Overview
This project analyzes the sales data of Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

### Data Sources

Data: The dataset used for this analysis: [city.csv](https://github.com/onatolumayowa/Coffee-Expansion-using-SQL/blob/main/city.csv), [customers.csv](https://github.com/onatolumayowa/Coffee-Expansion-using-SQL/blob/main/customers.csv), [sales.csv](https://github.com/onatolumayowa/Coffee-Expansion-using-SQL/blob/main/sales.csv), [products.csv](https://github.com/onatolumayowa/Coffee-Expansion-using-SQL/blob/main/products.csv)

### Tools

- MySQL - Exploratory Data Analysis
    - [Download here](https://dev.mysql.com/downloads/workbench/)

### Exploratory Data Analysis

EDA involved exploring the data to answer key questions such as:

- Coffee Consumers Count
  - How many people in each city are estimated to consume coffee, given that 25% of the population does?

- Total Revenue from Coffee Sales
  - What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?

- Sales Count for Each Product
  - How many units of each coffee product have been sold?

- Average Sales Amount per City
  - What is the average sales amount per customer in each city?

- City Population and Coffee Consumers
  - Provide a list of cities along with their populations and estimated coffee consumers.

- Top Selling Products by City
  - What are the top 3 selling products in each city based on sales volume?

- Customer Segmentation by City
  - How many unique customers are there in each city who have purchased coffee products?

- Average Sale vs Rent
  - Find each city and their average sale per customer and avg rent per customer

- Monthly Sales Growth
  - Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly).

- Market Potential Analysis
  - Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated coffee consumer.

### Data Analysis

Include some interesting code I worked with

- How many people in each city are estimated to consume coffee, given that 25% of the population does?
```sql
SELECT city_name, ROUND((population *0.25)/1000000, 2) AS coffee_consumers_in_millions,
	city_rank
FROM city 
ORDER BY 2 DESC;
```
- What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?
```sql
SELECT ci.city_name, Sum(total) AS total_revenue
FROM sales s
JOIN customers c USING(customer_id)
JOIN city ci ON ci.city_id = c.city_id
WHERE YEAR(s.sale_date) = 2023 AND EXTRACT(quarter FROM s.sale_date) = 4
GROUP BY 1
ORDER BY 2 DESC;
```
- How many units of each coffee product have been sold?
```sql
SELECT p.product_name, COUNT(s.sale_id) AS total_orders
FROM products p
LEFT JOIN sales s USING(product_id)
GROUP BY 1
ORDER BY 2 DESC;
```
- What is the average sales amount per customer in each city?
```sql
SELECT ci.city_name, SUM(s.total) AS total_revenue,
	COUNT(DISTINCT s.customer_id) AS total_customers,
	ROUND(SUM(s.total)/COUNT(DISTINCT s.customer_id), 2)AS avg_sale_per_customers
FROM sales s
JOIN customers c USING(customer_id)
JOIN city ci ON ci.city_id = c.city_id
GROUP BY 1
ORDER BY 2 DESC;
```
- Provide a list of cities along with their populations and estimated coffee consumers.
```sql
SELECT ci.city_name, 
	COUNT(DISTINCT cu.customer_id) AS unique_customers
FROM city ci
JOIN customers cu USING(city_id)
JOIN sales s ON cu.customer_id = s.customer_id
GROUP BY 1, 2;
```
- What are the top 3 selling products in each city based on sales volume?
```sql
WITH CTE_Example AS
(
SELECT ci.city_name, p.product_name, COUNT(s.sale_id) AS total_orders,
	DENSE_RANK() OVER(PARTITION BY ci.city_name ORDER BY COUNT(s.sale_id) DESC) AS rank_no
FROM sales s
JOIN products p USING(product_id)
JOIN customers cu USING(customer_id)
JOIN city ci ON cu.city_id = ci.city_id
GROUP BY 1, 2
)
SELECT *
FROM CTE_Example
WHERE rank_no <= 3;
```
- How many unique customers are there in each city who have purchased coffee products?
```sql
SELECT ci.city_name, COUNT(DISTINCT cu.customer_id) AS unique_customers
FROM city ci
JOIN customers cu USING(city_id)
JOIN sales s ON cu.customer_id = s.customer_id
JOIN products p ON s.product_id = p.product_id
GROUP BY 1;
```
- Find each city and their average sale per customer and avg rent per customer
```sql
WITH city_table AS
(
SELECT ci.city_name, COUNT(DISTINCT s.customer_id) AS total_customers,
	ROUND(SUM(s.total)/COUNT(DISTINCT s.customer_id), 2)AS avg_sale_per_customers
FROM sales s
JOIN customers c USING(customer_id)
JOIN city ci ON ci.city_id = c.city_id
GROUP BY 1
ORDER BY 2 DESC
),
city_rent AS
(
SELECT city_name, estimated_rent
FROM city
)
SELECT cr.city_name, cr.estimated_rent, ct.total_customers, ct.avg_sale_per_customers,
	ROUND(cr.estimated_rent/ct.total_customers, 2) AS avg_rent_per_customers
FROM city_rent cr
JOIN city_table ct ON cr.city_name = ct.city_name
ORDER BY 4 DESC;
```
- Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly).
```sql
WITH monthly_sales AS
(
SELECT ci.city_name, Month(sale_date) AS month, Year(sale_date) AS year,
	SUM(s.total) AS total_sale
FROM sales s
JOIN customers cu USING(customer_id)
JOIN city ci ON cu.city_id = ci.city_id
GROUP BY 1, 2, 3
ORDER BY 1, 3, 2
),
growth_ratio AS
(
SELECT city_name, month, year, total_sale AS current_month_sale, 
	LAG(total_sale, 1) OVER(PARTITION BY city_name ORDER BY year, month) AS last_month_sale
FROM monthly_sales
)
SELECT city_name, month, year, current_month_sale, last_month_sale, 
	ROUND((current_month_sale-last_month_sale)/last_month_sale * 100, 2) AS growth_ratio
FROM growth_ratio
WHERE last_month_sale IS NOT NULL;
```
- Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated coffee consumer.
```sql
WITH city_table AS
(
SELECT ci.city_name, SUM(s.total) AS total_revenue,
	COUNT(DISTINCT s.customer_id) AS total_customers,
	ROUND(SUM(s.total)/COUNT(DISTINCT s.customer_id), 2)AS avg_sale_per_customers
FROM sales s
JOIN customers c USING(customer_id)
JOIN city ci ON ci.city_id = c.city_id
GROUP BY 1
ORDER BY 2 DESC
),
city_rent AS
(
SELECT city_name, estimated_rent, ROUND((population * 0.25)/1000000, 2) AS estimated_coffee_consumer
FROM city
)
SELECT cr.city_name, total_revenue, cr.estimated_rent AS total_rent, ct.total_customers,
	estimated_coffee_consumer, ct.avg_sale_per_customers,
	ROUND(cr.estimated_rent/ct.total_customers, 2) AS avg_rent_per_customers
FROM city_rent cr
JOIN city_table ct ON cr.city_name = ct.city_name
ORDER BY 2 DESC;
```

### Findings

The analysis results are summarized as follows:
1. At the end of the analysis, I observed that Pune is the best performing category in terms of sales and revenue.
2. At the end of the analysis, I observed that the Highest number of customers by city_name is Delhi, which is 69.
3. At the end of the analysis, I also observed that Delhi have the Highest estimated coffee consumers at 7.7 million.

### Recommendations

After analyzing the data, the recommended top three cities for new store openings are:

- Pune

  - Average rent per customer is very low.
  - Highest total revenue.
  - Average sales per customer is also high.
    
- Delhi

  - Highest estimated coffee consumers at 7.7 million.
  - It has high total number of customers, which is 68.
  - Average rent per customer is 330 (still under 500).
    
- Jaipur

  - Highest number of customers, which is 69.
  - Average rent per customer is very low at 156.
  - Average sales per customer is better at 11.6k.
 
  ### References

  1. [sql by mosh](https://www.youtube.com/watch?v=t60b_MplxAA&list=PLOghUv2IDLKHKlkQNuzN8SPLYuVhhLlpa)
  2. [zero analyst](https://www.youtube.com/watch?v=ZZEP4ZRnDaU&list=PLF2u7Zn-dIxbeais0AkBxUqdWM1hnSJDS&index=9)









