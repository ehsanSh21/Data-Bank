# Data-Bank
SQL Case Study #4

### Introduction:

There is a new innovation in the financial industry called Neo-Banks: new aged digital only banks without physical branches.

Neo-Banks, digital-only banks without physical branches, have introduced a new era in the financial industry. Inspired by this innovation, Danny launches Data Bank, a digital bank with a secure distributed data storage platform.

Customers receive cloud data storage based on their account balances. To optimize growth and manage data storage effectively, the Data Bank team seeks assistance in analyzing metrics and data for informed forecasting and future development.

More information about this case study: https://8weeksqlchallenge.com/case-study-4/


### Entity Relationship Diagram: 

![alt text](https://github.com/ehsanSh21/Data-Bank/blob/main/case-study-4-erd.png)


### Case Study Questions

You can use the link of DB Fiddle below to easily access this datasets and use my queries!

<code class="language-plaintext highlighter-rouge">DB Fiddle link: </code> https://www.db-fiddle.com/f/2GtQz4wZtuNNu7zXH5HtV4/3


##### How many days on average are customers reallocated to a different node?

```sql
SELECT
avg(sub.date_diff) as avg_diff
from
(
SELECT
a.customer_id,a.region_id,a.node_id,a.start_date,a.end_date,
(a.end_date-a.start_date) as date_diff
FROM
data_bank.customer_nodes a
JOIN data_bank.customer_nodes b
ON a.customer_id = b.customer_id
AND a.end_date<b.start_date

WHERE b is not null

GROUP BY a.customer_id
,a.region_id,a.node_id,a.start_date,a.end_date
ORDER BY a.customer_id,a.start_date) as sub
;
```

###### output:

| avg_diff            |
| ------------------- |
| 14.6340000000000000 |



#### What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

```sql

WITH region_cte AS (

SELECT
sub.region_id,
count(*),
CEILING(count(*)/2) as median,
CEILING(0.8*count(*)) as eighty_percentile,
CEILING(0.95*count(*)) as ninety_five_percentile
from
(
SELECT
a.customer_id,a.region_id,a.node_id,a.start_date,a.end_date,
(a.end_date-a.start_date) as date_diff
FROM
data_bank.customer_nodes a
JOIN data_bank.customer_nodes b
ON a.customer_id = b.customer_id
AND a.end_date<b.start_date

WHERE b is not null

GROUP BY a.customer_id
,a.region_id,a.node_id,a.start_date,a.end_date
ORDER BY a.customer_id,a.start_date) as sub
GROUP BY sub.region_id
)

SELECT
sub3.region_id,
MAX(CASE WHEN med_id IS NOT NULL THEN date_diff END) AS median,
MAX(CASE WHEN eghit_id IS NOT NULL THEN date_diff END) AS
eighty_percentile,
MAX(CASE WHEN nine_id IS NOT NULL THEN date_diff END) AS ninety_five_percentile
FROM
(
SELECT
sub2.region_id,
sub2.date_diff,
med.region_id as med_id,
eight.region_id as eghit_id,
nine.region_id as nine_id
FROM
(
SELECT
*,
ROW_NUMBER() OVER (
       PARTITION BY region_id
       ORDER BY date_diff ASC
    ) as rn
from
(
SELECT
a.customer_id,a.region_id,a.node_id,a.start_date,a.end_date,
(a.end_date-a.start_date) as date_diff
FROM
data_bank.customer_nodes a
JOIN data_bank.customer_nodes b
ON a.customer_id = b.customer_id
AND a.end_date<b.start_date

WHERE b is not null

GROUP BY a.customer_id
,a.region_id,a.node_id,a.start_date,a.end_date
ORDER BY a.customer_id,a.start_date) as sub ) as sub2

left JOIN region_cte med ON sub2.region_id=med.region_id
AND sub2.rn=med.median

left JOIN region_cte eight ON sub2.region_id=eight.region_id
AND sub2.rn=eight.eighty_percentile

left JOIN region_cte nine ON sub2.region_id=nine.region_id
AND sub2.rn=nine.ninety_five_percentile

WHERE med is not null
OR eight is not null
or nine is not null) as sub3
GROUP BY sub3.region_id
;
```

###### output:

| region_id | median | eighty_percentile | ninety_five_percentile |
| --------- | ------ | ----------------- | ---------------------- |
| 1         | 15     | 23                | 28                     |
| 2         | 15     | 23                | 28                     |
| 3         | 15     | 24                | 28                     |
| 4         | 15     | 23                | 28                     |
| 5         | 15     | 24                | 28                     |





