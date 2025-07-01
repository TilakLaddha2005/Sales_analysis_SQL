#  Sales Analysis Using Intermediate SQL — Contoso 100K Dataset

This project focuses on performing **intermediate-level SQL analysis** on the Contoso 100K sales dataset. The goal is to uncover insights related to customer behavior, cohort analysis, revenue segmentation, and churn trends, using powerful SQL queries and visualizations.

---

## Project Overview

- **Tools Used**: MySQL / PostgreSQL (compatible),Dbeaver, Visual Studio Code
- **Dataset**: Contoso 100K sales data (imported into a SQL-compatible RDBMS)
- **Key Concepts Covered**:
  - Cohort analysis
  - Customer lifetime value (LTV)
  - Segmentation (high/mid/low value)
  - Churn rate calculation
  - Revenue trends by year

---

##  Summary of Findings, Business Insights & Actions

### ~VIEW --

```sql
CREATE OR REPLACE VIEW public.cohort_analysis AS
WITH customer_revenue AS (
         SELECT s.customerkey,
            s.orderdate,
            sum(s.quantity::double precision * s.netprice * s.exchangerate) AS total_net_revenue,
            count(s.orderkey) AS num_orders,
            c.countryfull,
            c.age,
            c.givenname,
            c.surname
           FROM sales s
             JOIN customer c ON c.customerkey = s.customerkey
          GROUP BY s.customerkey, s.orderdate, c.countryfull, c.age, c.givenname, c.surname
        )
 SELECT customerkey,
    orderdate,
    total_net_revenue,
    num_orders,
    countryfull,
    age,
    concat(TRIM(cr.givenname ),' ',TRIM(cr.surname )) AS clean_name,
    min(orderdate) OVER (PARTITION BY customerkey) AS first_purschase_date,
    EXTRACT(year FROM min(orderdate) OVER (PARTITION BY customerkey)) AS cohort_year
   FROM customer_revenue cr
   
```
---
### 1. Cohort Revenue Trends

- **Findings**: Peak revenue years: **2018–2019**; declining revenue/customer for newer cohorts.
- **Insight**: Older cohorts (2016–2019) are more profitable.
- **Actions**:
  - Prioritize retention in valuable cohorts.
  - Re-engage recent cohorts (post-2020) with tailored offers.

---

Query: 

```sql
SELECT 
ca.cohort_year,
COUNT(DISTINCT ca.customerkey) AS total_customers,
SUM(ca.total_net_revenue) AS total_revenue,
SUM(ca.total_net_revenue) / COUNT(DISTINCT ca.customerkey) AS customer_revenue
FROM 
cohort_analysis ca 
WHERE ca.orderdate = ca.first_purschase_date 
GROUP BY 
cohort_year 
```

---

##  Visualizations

![customer_revenue_by_cohort_year](/image/customer_revenue_by_cohort_year.png)


---


### 2. Customer Segmentation (LTV-Based)

Query:

```sql
WITH customer_ltv AS (
SELECT 
ca.customerkey ,
ca.clean_name ,
SUM(ca.total_net_revenue) AS total_ltv
FROM cohort_analysis ca 
GROUP BY 
customerkey ,
clean_name ),

customer_segment AS (
SELECT 
PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY total_ltv ) AS ltv_25th,
PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY total_ltv ) AS ltv_75th
FROM 
customer_ltv ),

segment_value AS (
SELECT 
c.*,
CASE 
	WHEN c.total_ltv < ltv_25th THEN '1-low value'
	WHEN c.total_ltv <= ltv_75th THEN '2-mid value' 
	ELSE '3-high value'
END AS customer_segment
FROM customer_ltv c,
customer_segment cs)

SELECT 
customer_segment,
sum(total_ltv) AS total_seg_ltv ,
COUNT(customerkey) AS total_customer,
sum(total_ltv) / COUNT(customerkey) AS avg_ltv
FROM
segment_value 
GROUP BY 
customer_segment

```

- **Findings**:
  - High-value segment (~12K customers) generates **>60% revenue**.
  - Low-value segment shows minimal contribution.
- **Insight**: Revenue is concentrated among a small, valuable base.
- **Actions**:
  - Focus on high-LTV retention and reward programs.
  - Upsell mid-value customers.
  - Limit low-value segment marketing unless profitable.

---

##  Visualizations

![customer_segmentation](/image/customer_segmentation.png)

---

### 3. Churn Trends

- **Findings**: 90%+ churn across all cohorts; only 8–10% active customers.
- **Insight**: Retention is a significant challenge.
- **Actions**:
  - Implement feedback loops, loyalty systems.
  - Automate re-engagement workflows.
  - Explore subscription/loyalty incentives.

---

Query:

```sql
WITH customer_last_purchase AS (
SELECT 
ca.customerkey ,
ca.clean_name ,
ca.orderdate ,
row_number() OVER (partition BY ca.customerkey ORDER BY ca.orderdate desc ) AS rn,
ca.first_purschase_date 
FROM 
cohort_analysis ca )

SELECT 
customerkey ,
clean_name ,
orderdate AS last_purchase_date,
CASE
	WHEN orderdate  < (SELECT max(orderdate) FROM sales)  - INTERVAL '6 months' THEN 'churned'
	ELSE 'active'
END AS customer_status
FROM 
customer_last_purchase 
WHERE rn = 1 AND 
first_purschase_date < (SELECT max(orderdate) FROM sales) - INTERVAL '6 months'

```
---
### 4. Customer Growth vs. Quality

- **Findings**: Surge in customer acquisition (2018–2019), but avg. revenue per customer declined post-2020.
- **Insight**: Recent growth may be sacrificing customer quality.
- **Actions**:
  - Analyze acquisition channels.
  - Optimize for high-LTV user acquisition.
  - Balance scale with profitability.

---

Query:

```sql
WITH customer_last_purchase AS (
SELECT 
ca.customerkey ,
ca.clean_name ,
ca.orderdate ,
row_number() OVER (partition BY ca.customerkey ORDER BY ca.orderdate desc ) AS rn,
ca.first_purschase_date,
ca.cohort_year 
FROM 
cohort_analysis ca )
,
customers_ch_ac AS (
SELECT 
customerkey ,
clean_name ,
orderdate AS last_purchase_date,
CASE
	WHEN orderdate  < (SELECT max(orderdate) FROM sales)  - INTERVAL '6 months' THEN 'churned'
	ELSE 'active'
END AS customer_status,
cohort_year 
FROM 
customer_last_purchase 
WHERE rn = 1 
AND first_purschase_date < (SELECT max(orderdate) FROM sales) - INTERVAL '6 months')

SELECT 
cohort_year ,
customer_status ,
COUNT(customerkey) AS num_customers,
sum(COUNT(customerkey)) OVER(PARTITION BY cohort_year) AS total_customers,
round(COUNT(customerkey)/sum(COUNT(customerkey)) OVER(PARTITION BY cohort_year),2) AS status_percentage
FROM
customers_ch_ac
GROUP BY
cohort_year ,
customer_status 

```
---

## Visualizations

![customer_retentation_by_cohort_year](/image/customer_retentation_by_cohort_year.png)


---
## Contact

**Tilak Laddha**  

📧 Email: [tilakladdhaofficial2005@gmail.com](mailto:tilakladdhaofficial2005@gmail.com)  

