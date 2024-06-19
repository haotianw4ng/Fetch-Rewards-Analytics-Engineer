# Fetch-Rewards-Analytics-Engineer

## First: Review Unstructured Data and Diagram a New Structured Relational Data Model

![58825b6b2f67ff068627f0872242a5f](https://github.com/haotianw4ng/Fetch-Rewards-Analytics-Engineer/assets/44035650/2959e0c0-7d72-43b1-8e77-f4f1a8724654)

```
Table users {
user_id varchar [primary key]
role varchar
sign_up_source varchar
state varchar
active boolean
created_date timestamp
last_login timestamp
}

Table brands {
brand_id varchar [primary key]
name varchar
category varchar
barcode varchar
top_brand boolean
cpg_id varchar
cpg_ref varchar
}

Table receipts {
receipt_id varchar [primary key]
user_id varchar
total_spent float
scanned_date timestamp
created_date timestamp
modified_date timestamp
finished_date timestamp
points_awarded_date timestamp
purchase_date timestamp
bonus_points_earned integer
bonus_points_earned_reason varchar
points_earned float
purchased_item_count integer
rewards_receipt_status varchar
}

Table items {
item_id varchar [primary key]
receipt_id varchar
barcode varchar
description varchar
final_price float
item_price float
quantity_purchased integer
needs_fetch_review boolean
user_flagged_barcode varchar
user_flagged_new_item boolean
user_flagged_price float
prevent_target_gap_points boolean
}

Ref: receipts.user_id > users.user_id // many-to-one
Ref: items.receipt_id > receipts.receipt_id // many-to-one
Ref: items.barcode > brands.barcode // many-to-one
```

## Second: Write queries that directly answer predetermined questions from a business stakeholder
1. What are the top 5 brands by receipts scanned for most recent month?
```
SELECT b.name, COUNT(DISTINCT r.receipt_id) AS receipt_count
FROM brands b
JOIN items i ON b.barcode = i.barcode
JOIN receipts r ON i.receipt_id = r.receipt_id
WHERE r.scanned_date BETWEEN DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
                          AND DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 day'
GROUP BY b.name
ORDER BY receipt_count DESC
LIMIT 5;
```

2. How does the ranking of the top 5 brands by receipts scanned for the recent month compare to the ranking for the previous month?
```
WITH CurrentMonth AS (
    SELECT b.name, COUNT(DISTINCT r.receipt_id) AS receipt_count
    FROM brands b
    JOIN items i ON b.barcode = i.barcode
    JOIN receipts r ON i.receipt_id = r.receipt_id
    WHERE r.scanned_date BETWEEN DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
                              AND DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 day'
    GROUP BY b.name
),
PreviousMonth AS (
    SELECT b.name, COUNT(DISTINCT r.receipt_id) AS receipt_count
    FROM brands b
    JOIN items i ON b.barcode = i.barcode
    JOIN receipts r ON i.receipt_id = r.receipt_id
    WHERE r.scanned_date BETWEEN DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '2 months'
                              AND DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month' - INTERVAL '1 day'
    GROUP BY b.name
)
SELECT cm.name AS current_month_brand,
       cm.receipt_count AS current_month_receipts,
       pm.receipt_count AS previous_month_receipts
FROM CurrentMonth cm
LEFT JOIN PreviousMonth pm ON cm.name = pm.name
ORDER BY cm.receipt_count DESC, pm.receipt_count DESC
LIMIT 5;
```
3. When considering average spend from receipts with 'rewardsReceiptStatus’ of ‘Accepted’ or ‘Rejected’, which is greater?
```
SELECT rewards_receipt_status, AVG(total_spent) AS average_spent
FROM receipts
WHERE rewards_receipt_status IN ('Accepted', 'Rejected')
GROUP BY rewards_receipt_status;
```
4. When considering total number of items purchased from receipts with 'rewardsReceiptStatus’ of ‘Accepted’ or ‘Rejected’, which is greater?
```
SELECT rewards_receipt_status, SUM(purchased_item_count) AS total_items_purchased
FROM receipts
WHERE rewards_receipt_status IN ('Accepted', 'Rejected')
GROUP BY rewards_receipt_status;
```
5. Which brand has the most spend among users who were created within the past 6 months?
```
SELECT b.name, SUM(r.total_spent) AS total_brand_spend
FROM brands b
JOIN items i ON b.barcode = i.barcode
JOIN receipts r ON i.receipt_id = r.receipt_id
JOIN users u ON r.user_id = u.user_id
WHERE u.created_date > CURRENT_DATE - INTERVAL '6 months'
GROUP BY b.name
ORDER BY total_brand_spend DESC
LIMIT 1;
```
6. Which brand has the most transactions among users who were created within the past 6 months?
```
SELECT b.name, COUNT(DISTINCT r.receipt_id) AS transaction_count
FROM brands b
JOIN items i ON b.barcode = i.barcode
JOIN receipts r ON i.receipt_id = r.receipt_id
JOIN users u ON r.user_id = u.user_id
WHERE u.created_date > CURRENT_DATE - INTERVAL '6 months'
GROUP BY b.name
ORDER BY transaction_count DESC
LIMIT 1;
```
