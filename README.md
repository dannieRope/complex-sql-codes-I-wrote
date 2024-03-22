# Complex sql codes I wrote so far

```sql
/*Find the top 5 customers who have made the highest total sales in each state, along with the product category they mostly purchased.
Use a Common Table Expression (CTE) to calculate the total sales for each customer in each state, and then retrieve the top 5 customers for each state.
Additionally, provide the product category that these top customers predominantly bought in each state.*/

WITH cte_customer AS (
                SELECT 
				      state,
					  Customer_Name,
					  category,
					  ROUND(SUM(Sales),2) AS total_sales,
					  ROW_NUMBER()OVER(PARTITION BY state ORDER BY SUM(Sales) DESC) AS sales_rank
				FROM Factsales F
				JOIN dim_customer c ON f.customerId = c.Customer_ID
				JOIN dim_product p ON f.productkey = p.productkey
				JOIN dim_geography g ON f.geographyKey = g.geographyKey
				GROUP BY state,Customer_Name,Category
)
SELECT State,
       Customer_Name,
	   total_sales
FROM cte_customer
WHERE sales_rank <= 5;
```

```sql
--Seasonal Sales Analysis: Identify the top-selling product category in each quarter of the year.
WITH topselling_cte AS (
                 SELECT CONCAT('Q',DATEPART(QUARTER,Orderdate)) AS quarters,
				 YEAR(Orderdate) AS Year,
				 Product_name,
				 SUM(Quantity) As qty,
				 ROW_NUMBER() OVER(PARTITION BY CONCAT('Q',DATEPART(QUARTER,Orderdate)),YEAR(Orderdate) ORDER BY SUM(Quantity)DESC) AS quarter_rank
				 FROM FactSales F
				 JOIN dim_product D ON F.ProductKey = D.ProductKey
				 GROUP BY CONCAT('Q',DATEPART(QUARTER,Orderdate)),YEAR(Orderdate),Product_name
)
SELECT quarters,
       product_name,
	   qty
FROM topselling_cte
WHERE quarter_rank = 1;
```

```sql
--Profitable Products Across Regions: Find the top 3 most profitable products in each region.
WITH ProfitableProducts AS (
    SELECT 
        Region,
        Product_Name,
        ROUND(SUM(profit),2) AS total_profit,
        ROW_NUMBER() OVER (PARTITION BY Region ORDER BY SUM(profit) DESC) AS product_rank
    FROM FactSales f
    JOIN  dim_product d ON f.ProductKey = d.ProductKey
    JOIN  dim_geography g ON f.GeographyKey = g.GeographyKey
    GROUP BY  Region, Product_Name
)
SELECT  Region,
        Product_Name,
        total_profit
FROM  ProfitableProducts
WHERE 
    product_rank <= 3;
```


```sql
/*Customer Segmentation: Segment customers into different groups (e.g., high-value, medium-value, low-value) based on their total purchases,
and analyze the average order value and frequency of each group.*/
WITH CustomerSegment AS (
    SELECT CustomerID,
           SUM(Sales) AS total_purchases
    FROM FactSales
    GROUP BY CustomerID
),
    Segmentation AS (
    SELECT CustomerID,
           total_purchases,
           CASE 
               WHEN total_purchases < 1000 THEN 'Low-Value' 
               WHEN total_purchases BETWEEN 1000 AND 10000 THEN 'Medium-Value' 
               ELSE 'High-Value' 
           END AS customer_segment
    FROM CustomerSegment
)
SELECT 
    customer_segment,
    COUNT(CustomerID) AS customer_count,
    ROUND(SUM(total_purchases),2) AS total_purchases,
    ROUND(AVG(total_purchases),2) AS average_order_value
FROM  Segmentation
GROUP BY customer_segment;
```

```sql
-- Customer Retention Rate: Determine the percentage of customers who made repeat purchases (more than one order) in 2017.

WITH Customer2017_cte AS (
    SELECT DISTINCT CustomerID
    FROM FactSales
    WHERE YEAR(OrderDate) = 2017
    GROUP BY CustomerID
    HAVING COUNT(Orderid) > 1 --all customers who made repeated purchases in 2017
),
allCustomer_cte AS (
    SELECT  DISTINCT CustomerID
    FROM FactSales
    WHERE YEAR(OrderDate) = 2017 --Total customers who made purchases in 2017
)

SELECT 
    ROUND(COUNT(c.CustomerID) * 100.0 / COUNT(a.CustomerID),0) as retention_rate
FROM Customer2017_cte c
JOIN allCustomer_cte a 
ON c.CustomerID = a.CustomerID;
```

