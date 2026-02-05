# SQL-Based Customer and Branch Performance Analysis for Retail Banking

## Business Context
A commercial bank operating multiple physical branches in different regions and an online banking system collects large volumes of customer transaction data. The retail banking analytics department requires structured analysis to evaluate customer activity across branches and digital channels.

## Data Challenge
Management currently cannot easily compare or get clear visibility into:
- Regional transaction performance  
- Customer usage of online banking services  
- Identification of high-value customers  
- Monitoring transaction growth trends by region  

Existing operational systems store transactional data but do not provide analytical insights for strategic decision-making.

## Expected Outcome
The analysis aims to:
- Identify top customers and high-performing branches  
- Measure regional transaction trends  
- Evaluate adoption of online banking services  
- Support data-driven decisions to improve customer engagement and digital banking growth  

## Success Criteria
- Identify the top 3 customers per branch using `RANK()`  
- Calculate running monthly totals using `SUM() OVER()`  
- Measure month-over-month growth using `LAG()`  
- Segment customers using `NTILE(4)`  
- Compute moving averages using `AVG() OVER()`

# Database Schema Design

## Overview
The schema captures transactional activities across physical branches and online banking. It is normalized to reduce redundancy and ensure relational integrity.

## Tables and Relationships

### BRANCHES
- Primary Key: branch_id  
- Columns: branch_name, region  

### CUSTOMERS
- Primary Key: customer_id  
- Foreign Key: branch_id → BRANCHES(branch_id)  
- Columns: full_name, registration_date  

### ACCOUNTS
- Primary Key: account_id  
- Foreign Key: customer_id → CUSTOMERS(customer_id)  
- Columns: account_type, open_date  

### TRANSACTIONS
- Primary Key: transaction_id  
- Foreign Key: account_id → ACCOUNTS(account_id)  
- Columns: transaction_date, amount, channel_type (ONLINE / BRANCH)

## ER Diagram

![ER Diagram](Screenshots/ER%20diagram.png)

# SQL Joins and Implementations


## INNER JOIN
```sql
-- Retrieve all transactions along with customer name and branch
SELECT 
    t.transaction_id,
    c.full_name AS customer_name,
    b.branch_name,
    t.transaction_date,
    t.amount,
    t.channel_type
FROM TRANSACTIONS t
INNER JOIN ACCOUNTS a 
    ON t.account_id = a.account_id       -- link transaction to account
INNER JOIN CUSTOMERS c 
    ON a.customer_id = c.customer_id     -- link account to customer
INNER JOIN BRANCHES b 
    ON c.branch_id = b.branch_id         -- link customer to branch
ORDER BY t.transaction_date;

```
![INNER JOIN](Screenshots/SQL%20JOINS/1.INNER%20JOIN.png)

This query retrieves all transactions together with the customer who performed them and the branch they belong to. It helps management track branch-level activity and transaction volumes.

## LEFT JOIN
```sql
-- List all customers even if they have no transactions
SELECT 
    c.full_name AS customer_name,
    b.branch_name,
    t.transaction_id,
    t.amount
FROM CUSTOMERS c
LEFT JOIN ACCOUNTS a 
    ON c.customer_id = a.customer_id     -- link customer to account
LEFT JOIN TRANSACTIONS t 
    ON a.account_id = t.account_id       -- link account to transactions
LEFT JOIN BRANCHES b 
    ON c.branch_id = b.branch_id         -- link customer to branch
WHERE t.transaction_id IS NULL;           -- filter customers with no transactions

```
![LEFT INNER JOIN](Screenshots/SQL%20JOINS/2.LEFT%20INNER%20JOIN.png)

This identifies inactive customers who have not yet performed any transactions. The bank can target these customers with promotions or engagement campaigns.

## RIGHT JOIN
```sql
-- Display all transactions and the associated customer names
SELECT 
    c.full_name,
    t.transaction_id,
    t.amount
FROM CUSTOMERS c
RIGHT JOIN ACCOUNTS a 
    ON c.customer_id = a.customer_id
RIGHT JOIN TRANSACTIONS t 
    ON a.account_id = t.account_id;
```
![RIGHT INNER JOIN](Screenshots/SQL%20JOINS/3.RIGHT%20INNER%20JOIN.png)

This query ensures that every transaction is displayed even if some customer information is missing, though for our case no information is missing. It helps the bank verify transaction completeness and detect possible data inconsistencies. 


## FULL OUTER JOIN
```sql
-- Combine all customers and transactions to get full overview
SELECT 
    c.full_name AS customer_name,
    a.account_id,
    t.transaction_id,
    t.amount
FROM CUSTOMERS c
FULL OUTER JOIN ACCOUNTS a 
    ON c.customer_id = a.customer_id
FULL OUTER JOIN TRANSACTIONS t 
    ON a.account_id = t.account_id
ORDER BY c.full_name;
```
![FULL INNER JOIN](Screenshots/SQL%20JOINS/4.FULL%20INNER%20JOIN.png)

This rovides a full overview of all customers, including those with and without transactions. This helps the bank reconcile accounts and understand overall customer activity.


## SELF JOIN

```sql
-- Find customers who registered on the same date
SELECT 
    c1.full_name AS customer_1,
    c2.full_name AS customer_2,
    c1.registration_date
FROM CUSTOMERS c1
JOIN CUSTOMERS c2
    ON c1.registration_date = c2.registration_date
    AND c1.customer_id < c2.customer_id;
```
![SELF JOIN](Screenshots/SQL%20JOINS/5.SELF%20JOIN.png)

This query identifies customers who registered on the same day. The bank can use this information to analyze customer acquisition patterns or evaluate the effectiveness of specific marketing campaigns launched on certain dates.

# Window functions implementations

## Ranking function

```sql
-- Rank customers within each branch based on total transaction value
SELECT *
FROM (
    SELECT
        b.branch_name,
        c.full_name,
        SUM(t.amount) AS total_amount,
        RANK() OVER (
            PARTITION BY b.branch_name
            ORDER BY SUM(t.amount) DESC
        ) AS rank_no
    FROM TRANSACTIONS t
    JOIN ACCOUNTS a ON t.account_id = a.account_id
    JOIN CUSTOMERS c ON a.customer_id = c.customer_id
    JOIN BRANCHES b ON c.branch_id = b.branch_id
    GROUP BY b.branch_name, c.full_name
)
WHERE rank_no <= 3;
```
![Ranking function](Screenshots/SQL%20Window%20functions/1.Ranking%20funnction.png)

This query ranks customers within each branch according to their total transaction value. It helps the bank identify high-value customers in each region. These customers can be targeted for premium services and loyalty programs.

## Aggregate function
```sql
-- Running monthly transaction totals per branch
SELECT
    b.branch_name,
    TRUNC(t.transaction_date, 'MM') AS month,
    SUM(t.amount) AS monthly_total,
    SUM(SUM(t.amount)) OVER (
        PARTITION BY b.branch_name
        ORDER BY TRUNC(t.transaction_date,'MM')
    ) AS running_total
FROM TRANSACTIONS t
JOIN ACCOUNTS a ON t.account_id = a.account_id
JOIN CUSTOMERS c ON a.customer_id = c.customer_id
JOIN BRANCHES b ON c.branch_id = b.branch_id
GROUP BY b.branch_name, TRUNC(t.transaction_date,'MM')
ORDER BY b.branch_name, month;
```
![Aggregate window function](Screenshots/SQL%20Window%20functions/2.Aggregare%20window%20funnction.png)


This query calculates transaction totals in different months for each branch. It allows management to monitor financial growth trends continuously. This helps in forecasting revenue performance.

## Navigation function

```sql
-- Compare current month total with previous month
SELECT
    branch_name,
    month,
    monthly_total,
    monthly_total -
    LAG(monthly_total) OVER (
        PARTITION BY branch_name
        ORDER BY month
    ) AS monthly_growth
FROM (
    SELECT
        b.branch_name,
        TRUNC(t.transaction_date,'MM') AS month,
        SUM(t.amount) AS monthly_total
    FROM TRANSACTIONS t
    JOIN ACCOUNTS a ON t.account_id = a.account_id
    JOIN CUSTOMERS c ON a.customer_id = c.customer_id
    JOIN BRANCHES b ON c.branch_id = b.branch_id
    GROUP BY b.branch_name, TRUNC(t.transaction_date,'MM')
);
```
![Navigate function](Screenshots/SQL%20Window%20functions/3.Navigate%20funnction.png)


This query compares each month’s transaction amount with the previous month using the LAG function. It helps management measure growth or decline in branch performance. This insight supports timely operational decisions.

## Distribution function

```sql
-- Divide customers into 4 value groups
SELECT
    c.full_name,
    SUM(t.amount) AS total_amount,
    NTILE(4) OVER (
        ORDER BY SUM(t.amount) DESC
    ) AS quartile
FROM TRANSACTIONS t
JOIN ACCOUNTS a ON t.account_id = a.account_id
JOIN CUSTOMERS c ON a.customer_id = c.customer_id
GROUP BY c.full_name;
```
![Distribution function](Screenshots/SQL%20Window%20functions/4.Distribution%20funnction.png)


This query segments customers into four groups by total transaction value, with the first quartile as the highest-value and the fourth quartile as the lowest. It helps the bank target marketing and focus on retaining top customers.

# Results analysis

### Descriptive (What happened?)

- Branches show varying transaction volumes each month.
- Some branches have very high single-month totals, while others show lower activity.
- Customer segmentation reveals high-value vs low-value customers, and some customers have not made any transactions.
- Ranking shows the top-performing customers per branch.

### Diagnostic (Why did it happen?)

- Variations in branch totals occur due to irregular transaction activity — some months have few transactions.
- Customer quartiles reflect the differences in individual transaction behavior, not the number of transactions.
- Declines in month-over-month growth appear when months with no transactions are compared to active months.
- Top customers per branch drive most of the transaction value, which indicates concentration of activity among a few customers.

### Prescriptive (What should be done?)

- Target low-activity or inactive customers (found via LEFT JOIN) with promotions or digital banking campaigns.
- Focus on top customers for premium services and retention programs.
- Use moving averages and running totals to forecast branch performance and plan staffing or marketing.
- Investigate branches with high fluctuations to understand seasonal or operational issues.

# REFERENCE

- Oracle Corporation. (2025). Oracle Database SQL Language Reference. Oracle Documentation. https://docs.oracle.com/en/database/oracle/oracle-database/
- L. Groff, J. Weinberg, & P. Oppel. (2022). SQL for Data Analysis: Advanced Techniques for Data-Driven Decision Making. O’Reilly Media.
- Silberschatz, A., Korth, H., & Sudarshan, S. (2020). Database System Concepts (7th ed.). McGraw-Hill Education.
- IBM Analytics. (2021). Using SQL and Data Analytics in Retail Banking. IBM Knowledge Center. https://www.ibm.com/analytics/retail-banking
- SQL Window Function | How to write SQL Query using RANK, DENSE RANK, LEAD/LAG | SQL Queries Tutorial. https://youtu.be/Ww71knvhQ-s?si=C2TAPEKvBNjYHHc2

##
“All sources were properly cited. Implementations and analysis represent original work. No AI generated content was copied without attribution or adaptation.”


