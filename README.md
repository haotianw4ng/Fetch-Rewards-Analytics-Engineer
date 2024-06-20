# Fetch-Rewards-Analytics-Engineer

## First: Review Unstructured Data and Diagram a New Structured Relational Data Model
The code that I used to review the data structure is in json_analysis.ipynb

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
```sql
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
```sql
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
```sql
SELECT rewards_receipt_status, AVG(total_spent) AS average_spent
FROM receipts
WHERE rewards_receipt_status IN ('Accepted', 'Rejected')
GROUP BY rewards_receipt_status;
```
4. When considering total number of items purchased from receipts with 'rewardsReceiptStatus’ of ‘Accepted’ or ‘Rejected’, which is greater?
```sql
SELECT rewards_receipt_status, SUM(purchased_item_count) AS total_items_purchased
FROM receipts
WHERE rewards_receipt_status IN ('Accepted', 'Rejected')
GROUP BY rewards_receipt_status;
```
5. Which brand has the most spend among users who were created within the past 6 months?
```sql
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
```sql
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

## Third: Evaluate Data Quality Issues in the Data Provided
The code that I used to conduct the analysis is in **`json_analysis.ipynb`**
### **1. Users.json**

#### Missing Values
- **`lastLogin`:** 62 missing entries
- **`signUpSource`:** 48 missing entries
- **`state`:** 56 missing entries

#### Duplicate Rows
- **Total Duplicates:** 283 duplicate records identified

#### Data Structure
- Complex nested JSON structures found in:
  - **`_id`**
  - **`createdDate`**
  - **`lastLogin`**

#### Data Types
- Mostly object types, except for `active` which is boolean.

### **2. Brands.json**

#### Missing Values
- **`category`:** 155 missing entries
- **`categoryCode`:** 650 missing entries
- **`topBrand`:** 612 missing entries
- **`brandCode`:** 234 missing entries

#### Duplicate Rows
- **Total Duplicates:** No duplicates found

#### Data Structure
- Complex nested JSON structures found in:
  - **`_id`**
  - **`cpg`**

#### Data Types
- All fields stored as object types.

### **3. Receipts.json**

#### Missing Values
- **`bonusPointsEarned` and `bonusPointsEarnedReason`:** 575 missing entries each
- **`finishedDate`:** 551 missing entries
- **`pointsAwardedDate`:** 582 missing entries
- **`pointsEarned`:** 510 missing entries
- **`purchaseDate`:** 448 missing entries
- **`purchasedItemCount`:** 484 missing entries
- **`totalSpent`:** 435 missing entries

#### Duplicate Rows
- **Total Duplicates:** No duplicates detected

#### Data Structure
- Complex nested JSON structures found in:
  - **`_id`**
  - **`createDate`**
  - **`dateScanned`**
  - **`finishedDate`**
  - **`modifyDate`**
  - **`pointsAwardedDate`**
  - **`purchaseDate`**
  - **`rewardsReceiptItemList`**

#### Data Types
- A mix of `object` and `float64` types.

## General Observations

The datasets show considerable issues with missing data, particularly in `receipts.json`, which could impact the accuracy of analytics. The `users.json` dataset has a significant number of duplicates that need addressing to ensure data integrity. Complex data structures in all datasets suggest a need for data transformation to simplify analysis and processing.

## Recommendations

1. **Data Cleaning:** Prioritize resolving duplicates in `users.json` and addressing extensive missing values in `receipts.json`.
2. **Data Structure Optimization:** Simplify nested JSON structures to enhance usability and compatibility with data processing tools.
3. **Enhanced Data Collection Practices:** Review and improve data collection methods to reduce the occurrence of missing and inconsistent data.



## Fourth: Communicate with Stakeholders
```
Hi [Stakeholder’s Name],

I hope this message finds you well. I have completed a preliminary review of our primary datasets: Users, Brands, and Receipts. This analysis aimed to identify data quality issues that could impact our operations and decision-making.

During my review, several critical issues came to light:

1. Completeness: There are noticeable gaps with missing data in key fields across all datasets. These gaps could potentially skew our analysis and business insights.
2. Uniqueness: Duplicate records were found in our databases, which could lead to overestimations or erroneous reporting in our performance metrics.
3. Consistency: I observed inconsistencies in data types and formats that suggest misalignment in how data entries are logged across different platforms.
4. Usability: Some datasets contain complex nested structures, complicating their use and integration in our current analytical processes.

To address these issues effectively, I need further information:

1. Data Collection Methods: Understanding how data is collected will help us pinpoint the root causes of inconsistencies and duplicates.
2. Business Rules: Clarity on business rules for data validation is crucial for ensuring the accuracy of our analysis.
3. Recent Changes: Any recent changes in how data is recorded could explain some of the anomalies observed.

Additionally, as our data volume grows, I anticipate potential performance bottlenecks. Implementing robust data infrastructure such as distributed databases or optimizing our current data processing methods might be necessary.

Could we schedule a time to discuss how we might improve our data collection and processing to enhance data quality? Your insights would be invaluable

Thank you for your attention to this matter. I look forward to your guidance.

Best,

Haotian Wang
```
