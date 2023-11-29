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


##### For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

```sql
SELECT
sub.transaction_month,sub.transaction_year,COUNT(*)
FROM
(
SELECT
   EXTRACT(MONTH FROM txn_date) AS transaction_month,
   EXTRACT(YEAR FROM txn_date) AS transaction_year,
customer_id,
COUNT(*) as total_txn,
sum(case when txn_type= 'deposit' then 1 else 0 end)as depo_count,
sum(case when txn_type= 'withdrawal' then 1 else 0 end)as with_count,
sum(case when txn_type= 'purchase' then 1 else 0 end)as purch_count
FROM
data_bank.customer_transactions
GROUP BY customer_id, EXTRACT(MONTH FROM txn_date),
   EXTRACT(YEAR FROM txn_date)
ORDER BY EXTRACT(MONTH FROM txn_date) asc ,total_txn desc) as sub
WHERE sub.depo_count > 1
AND ( sub.with_count > 0 OR sub.purch_count > 0 )
GROUP BY sub.transaction_month,sub.transaction_year
ORDER BY sub.transaction_month
;
```

##### output:

| transaction_month | transaction_year | count |
| ----------------- | ---------------- | ----- |
| 1                 | 2020             | 168   |
| 2                 | 2020             | 181   |
| 3                 | 2020             | 192   |
| 4                 | 2020             | 70    |


##### Data Allocation Challenge: running customer balance column that includes the impact each transaction

```sql
WITH trans_cte AS (
select
*,
  ROW_NUMBER() OVER (
       PARTITION BY customer_id
       ORDER BY txn_date
    ) rn
FROM
data_bank.customer_transactions
),

sum_sub_running AS(
SELECT
sub.customer_id,sub.txn_date,sub.rn,sub.txn_type,sub.txn_amount,sub.a_type_add,
sum(case WHEN sub.b_type_add = 1 then sub.b_amount ELSE 0 end) as
sum_adds,
sum(case WHEN sub.b_type_add = 0 then sub.b_amount ELSE 0 end) as
sub_adds
FROM
(
SELECT
a.*,
b.txn_type as b_type,
(case when b.txn_amount is not NULL then b.txn_amount else 0 end) as
b_amount,
(case when a.txn_type = 'deposit' then 1 ELSE 0 END ) as a_type_add,
(case when b.txn_type = 'deposit' then 1 ELSE 0 END ) as b_type_add
FROM
trans_cte a
left JOIN trans_cte b
ON a.customer_id = b.customer_id
AND a.rn > b.rn) as sub

GROUP BY
sub.customer_id,sub.txn_date,sub.rn,sub.txn_type,sub.txn_amount,sub.a_type_add
ORDER BY sub.customer_id,sub.rn)

SELECT
*,
(CASE
     WHEN a_type_add = 1 THEN sum_adds - sub_adds + txn_amount
     ELSE sum_adds - sub_adds - txn_amount
   END) AS running_customer_balance
FROM
sum_sub_running
ORDER BY sum_sub_running,customer_id,sum_sub_running.txn_date
;
```

##### sample output:

| customer_id | txn_date                 | rn  | txn_type | txn_amount | a_type_add | sum_adds | sub_adds | running_customer_balance |
| ----------- | ------------------------ | --- | -------- | ---------- | ---------- | -------- | -------- | ------------------------ |
| 1           | 2020-01-02T00:00:00.000Z | 1   | deposit  | 312        | 1          | 0        | 0        | 312                      |
| 1           | 2020-03-05T00:00:00.000Z | 2   | purchase | 612        | 0          | 312      | 0        | -300                     |
| 1           | 2020-03-17T00:00:00.000Z | 3   | deposit  | 324        | 1          | 312      | 612      | 24                       |
| 1           | 2020-03-19T00:00:00.000Z | 4   | purchase | 664        | 0          | 636      | 612      | -640                     |
| 2           | 2020-01-03T00:00:00.000Z | 1   | deposit  | 549        | 1          | 0        | 0        | 549                      |
| 2           | 2020-03-24T00:00:00.000Z | 2   | deposit  | 61         | 1          | 549      | 0        | 610                      |
| 3           | 2020-01-27T00:00:00.000Z | 1   | deposit  | 144        | 1          | 0        | 0        | 144                      |




