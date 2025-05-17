-- Assessment_Q1.sql
-- Identify customers with at least one funded savings and one funded investment plan

SELECT 
    u.id AS owner_id,
    CONCAT(u.first_name, ' ', u.last_name) AS name,
    COUNT(DISTINCT s.id) AS savings_count,
    COUNT(DISTINCT p.id) AS investment_count,
    SUM(s.confirmed_amount) / 100 AS total_deposits -- convert kobo to naira
FROM users_customuser u
LEFT JOIN savings_savingsaccount s ON u.id = s.owner_id AND s.is_regular_savings = 1 AND s.confirmed_amount > 0
LEFT JOIN plans_plan p ON u.id = p.owner_id AND p.is_a_fund = 1 AND p.confirmed_amount > 0
GROUP BY u.id, u.first_name, u.last_name
HAVING COUNT(DISTINCT s.id) > 0 AND COUNT(DISTINCT p.id) > 0
ORDER BY total_deposits DESC;


-- Assessment_Q2.sql
-- Categorize customers by average number of monthly transactions

WITH transactions_per_customer AS (
    SELECT 
        owner_id,
        COUNT(*) AS total_txns,
        (DATE_PART('year', MAX(created_at)) - DATE_PART('year', MIN(created_at))) * 12 +
        (DATE_PART('month', MAX(created_at)) - DATE_PART('month', MIN(created_at))) + 1 AS months_active
    FROM savings_savingsaccount
    GROUP BY owner_id
),
avg_txns AS (
    SELECT 
        owner_id,
        total_txns,
        months_active,
        total_txns * 1.0 / months_active AS avg_txns_per_month
    FROM transactions_per_customer
),
categorized AS (
    SELECT 
        CASE 
            WHEN avg_txns_per_month >= 10 THEN 'High Frequency'
            WHEN avg_txns_per_month BETWEEN 3 AND 9 THEN 'Medium Frequency'
            ELSE 'Low Frequency'
        END AS frequency_category,
        COUNT(*) AS customer_count,
        ROUND(AVG(avg_txns_per_month), 1) AS avg_transactions_per_month
    FROM avg_txns
    GROUP BY frequency_category
)
SELECT * FROM categorized;


-- Assessment_Q3.sql
-- Find accounts with no inflow transactions in the last 365 days

SELECT 
    id AS plan_id,
    owner_id,
    'Savings' AS type,
    MAX(created_at) AS last_transaction_date,
    DATE_PART('day', CURRENT_DATE - MAX(created_at)) AS inactivity_days
FROM savings_savingsaccount
WHERE confirmed_amount > 0
GROUP BY id, owner_id
HAVING MAX(created_at) < CURRENT_DATE - INTERVAL '365 days'

UNION ALL

SELECT 
    id AS plan_id,
    owner_id,
    'Investment' AS type,
    MAX(created_at) AS last_transaction_date,
    DATE_PART('day', CURRENT_DATE - MAX(created_at)) AS inactivity_days
FROM plans_plan
WHERE is_a_fund = 1 AND confirmed_amount > 0
GROUP BY id, owner_id
HAVING MAX(created_at) < CURRENT_DATE - INTERVAL '365 days';


-- Assessment_Q4.sql
-- Estimate Customer Lifetime Value (CLV)

WITH txn_summary AS (
    SELECT 
        s.owner_id,
        COUNT(*) AS total_txns,
        SUM(s.confirmed_amount) / 100 AS total_value,
        MIN(u.date_joined) AS joined,
        DATE_PART('year', CURRENT_DATE) * 12 + DATE_PART('month', CURRENT_DATE) -
        (DATE_PART('year', MIN(u.date_joined)) * 12 + DATE_PART('month', MIN(u.date_joined))) AS tenure_months
    FROM savings_savingsaccount s
    JOIN users_customuser u ON u.id = s.owner_id
    GROUP BY s.owner_id
),
clv_calc AS (
    SELECT 
        owner_id AS customer_id,
        (SELECT CONCAT(first_name, ' ', last_name) FROM users_customuser WHERE id = t.owner_id) AS name,
        tenure_months,
        total_txns,
        ROUND((total_txns * 0.001 / NULLIF(tenure_months, 0)) * 12, 2) AS estimated_clv
    FROM txn_summary t
)
SELECT * FROM clv_calc
ORDER BY estimated_clv DESC;
