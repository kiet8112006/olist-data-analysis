# olist-data-analysis
SQL Data Quality Analysis for Olist E-commerce Dataset
## Dataset Overview

The dataset contains transactional data from the Brazilian
e-commerce platform Olist.

It includes information about customers, orders, order items,
payments, reviews, products, sellers, geolocation, and product
category translations.

The dataset contains approximately **100,000 orders between
2016 and 2018**, covering multiple online marketplaces.

---

## Data Quality Dimensions

The dataset was evaluated using several common data quality dimensions:

- **Data Completeness** – detecting missing values
- **Data Uniqueness** – identifying duplicate records
- **Data Validity** – verifying acceptable value ranges
- **Data Consistency** – validating logical relationships between fields
---
### Core Transaction Flow

The primary transaction flow of the dataset is:

customers → orders → order_items → products

This flow represents the lifecycle of an e-commerce transaction,
starting from customer purchase to the product level.

## Data Model
Understanding the relationships between tables is essential for:

- performing accurate joins between tables
- validating **foreign key relationships**
- detecting potential **data quality issues**
- supporting downstream analytical queries

## Analysis Scope

This project focuses on evaluating the data quality of
each table in the dataset.

The analysis includes:

- missing value detection
- duplicate record identification
- logical validation of timestamps
- validation of numeric fields
- cross-table consistency checks

### Table Relationships

| Table | Key Column | Related Table | Relationship |
|------|------|------|------|
| customers | customer_id | orders | One customer can place multiple orders |
| orders | order_id | order_items | One order can contain multiple items |
| order_items | product_id | products | Each order item references a product |
| order_items | seller_id | sellers | Each item is fulfilled by a seller |
| orders | order_id | order_payments | Each order can have one or more payments |
| orders | order_id | order_reviews | Each order can have a customer review |
| customers | customer_zip_code_prefix | geolocation | Used to link customer location |
| products | product_category_name | product_category_name_translation | Translates Portuguese category names to English |

### Simplified Schema

customers  
→ orders  
→ order_items  
→ products  

Additional supporting tables:

- sellers
- order_payments
- order_reviews
- geolocation
- product_category_name_translation

### Purpose of the Data Model

Understanding these relationships allows us to:

- perform accurate joins between tables
- validate foreign key relationships
- detect potential data quality issues
- support downstream analytical queries

## Table Analysis

### 1. Customers Table

#### Data Completeness

To evaluate data completeness, all columns in the `olist_customers_dataset` table were checked for missing values.

SQL queries used:

```sql
SELECT
SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS null_customer_id,
SUM(CASE WHEN customer_unique_id IS NULL THEN 1 ELSE 0 END) AS null_customer_unique_id,
SUM(CASE WHEN customer_zip_code_prefix IS NULL THEN 1 ELSE 0 END) AS null_zip,
SUM(CASE WHEN customer_city IS NULL THEN 1 ELSE 0 END) AS null_city,
SUM(CASE WHEN customer_state IS NULL THEN 1 ELSE 0 END) AS null_state
FROM dbo.olist_customers_dataset;
```


| Column | NULL count |
|------|------|
| customer_id | 0 |
| customer_unique_id | 0 |
| customer_zip_code_prefix | 0 |
| customer_city | 0 |
| customer_state | 0 |

Conclusion:  
No missing values were detected in the customers table.

---
#### Duplicate Check

To ensure data integrity, the `customer_id` column was checked for duplicate values.

SQL query used:

```sql
SELECT customer_id, COUNT(*) AS duplicated_customer_id
FROM dbo.olist_customers_dataset
GROUP BY customer_id
HAVING COUNT(*) > 1;
```
Result:

The query returned **0 rows**, which means no `customer_id` values appear more than once.

Conclusion:
The `customer_id` column is unique across the dataset and can safely be used as the primary key of the customers table
#### Customer Unique ID Analysis

To understand customer purchasing behavior, the number of orders associated with each `customer_unique_id` was analyzed.

SQL query used:

```sql
select count(distinct customer_id) as order_counts, customer_unique_id
from dbo.olist_customers_dataset
group by customer_unique_id
having count(distinct customer_id) > 1
order by order_counts desc;
```
Result:

The analysis shows that several `customer_unique_id` values are associated with multiple orders.

For example:

- The highest number of orders placed by a single customer is **17 orders**.
- Other customers placed **9, 7, and 6 orders**.

Conclusion:

This indicates that some customers made **repeat purchases** on the Olist platform.

Therefore:

- `customer_id` represents a **unique order record**
- `customer_unique_id` represents the **actual customer**

This structure is consistent with a typical **e-commerce database design**, where a single customer can place multiple orders.
This design allows the dataset to track multiple orders made by the same customer across time.
#### ZIP Code Standardization

The column customer_zip_code_prefix represents the customer's postal code prefix rather than the full ZIP code.
To ensure consistency, the ZIP code format was standardized to **5 digits**.

SQL query used:

```sql
update dbo.olist_customers_dataset
set customer_zip_code_prefix = right('00000' + customer_zip_code_prefix, 5)
where customer_zip_code_prefix is not null;
```
Result:

All ZIP code prefixes were standardized to a **5-digit format** by padding leading zeros when necessary.

Conclusion:

The `customer_zip_code_prefix` column now follows a consistent **5-digit postal code format**, ensuring compatibility when joining with the geolocation dataset.
#### Text Standardization

To ensure consistency in text-based columns, the `customer_city` and `customer_state` fields were standardized by:

- Removing leading and trailing spaces
- Converting all text to uppercase format

SQL queries:

```sql
update dbo.olist_customers_dataset
set customer_city = upper(ltrim(rtrim(customer_city)))
where customer_city is not null;

update dbo.olist_customers_dataset
set customer_state = upper(ltrim(rtrim(customer_state)))
where customer_state is not null;
```
Result:

All values in `customer_city` and `customer_state` were standardized by removing extra spaces and converting the text to uppercase format.

Conclusion:

This ensures consistent formatting of location data and prevents inconsistencies during grouping, filtering, or aggregation in future analysis.
This prevents inconsistencies such as:
- "Sao Paulo"
- "sao paulo"
- "SAO PAULO"
#### Geolocation Validation

To verify that customer ZIP codes can be linked to geographic coordinates,  
the `customer_zip_code_prefix` column was compared with the geolocation dataset.

The geolocation table was previously aggregated to create an average latitude and longitude for each ZIP code prefix (`geolocation_avg`).

SQL query used:

```sql
select 
    c.customer_id,
    c.customer_unique_id,
    c.customer_zip_code_prefix,
    c.customer_city,
    c.customer_state,
    g.avg_lat,
    g.avg_lng,
    case 
        when g.geolocation_zip_code_prefix is null then 1
        else 0
    end as flag_missing_geo
into dbo.olist_customers_geo_check_dataset
from dbo.olist_customers_dataset c
left join dbo.geolocation_avg g
    on c.customer_zip_code_prefix = g.geolocation_zip_code_prefix;
```
Result:

The validation identified **278 customer records** whose ZIP code prefixes could not be matched with the geolocation dataset.

For these records:

- `avg_lat` and `avg_lng` are NULL
- `flag_missing_geo = 1`

Conclusion:
This validation confirms that some customer ZIP code prefixes do not exist in the geolocation reference table.
These records may require additional investigation during geographic analysis.  

#### Geolocation Validation Impact Analysis
To measure the impact of unmatched ZIP codes, the proportion of customers whose ZIP codes could not be linked to the geolocation dataset was calculated.

SQL query used:

```sql
SELECT 
    COUNT(*) AS total_customers,
    SUM(CASE WHEN flag_missing_geo = 1 THEN 1 ELSE 0 END) AS missing_geo_count,
    ROUND(
        100.0 * SUM(CASE WHEN flag_missing_geo = 1 THEN 1 ELSE 0 END) / COUNT(*),
        2
    ) AS missing_geo_percentage
FROM dbo.olist_customers_geo_check_dataset;
```
Result:

| Metric | Value |
|------|------|
| Total customers | 99,441 |
| Missing geolocation ZIP codes | 278 |
| Percentage of affected records | 0.28% |

Conclusion:

Out of **99,441 customers**, **278 ZIP codes** could not be matched with the geolocation dataset.  
This represents approximately **0.28% of the total records**.

Since the percentage is very small, the issue is considered **minor** and does not significantly impact geographic analysis.

These records were **retained in the dataset** and marked with `flag_missing_geo = 1` to indicate missing geographic information.

### Customers Table Data Quality Summary
| Check | Result |
|------|------|
Missing values | None detected |
Duplicate customer_id | None detected |
Repeat customers | Detected |
ZIP-code match with geolocation | 99.72% matched |
Data standardization | Applied |

### 2. Geolocation Data Aggregation

The original `olist_geolocation_dataset` contains multiple records for the same ZIP code prefix because each record represents a specific latitude and longitude point.

To create a cleaner reference table for geographic joins, the geolocation data was aggregated by ZIP code prefix.

SQL query used:

```sql
SELECT 
    geolocation_zip_code_prefix,
    AVG(geolocation_lat) AS avg_lat,
    AVG(geolocation_lng) AS avg_lng,
    MAX(geolocation_city) AS city,
    MAX(geolocation_state) AS state
INTO geolocation_avg
FROM dbo.olist_geolocation_dataset
GROUP BY geolocation_zip_code_prefix;
```
Result:

The query created a new table called `geolocation_avg` where each `geolocation_zip_code_prefix` appears only once.

For each ZIP code prefix:
- `avg_lat` and `avg_lng` represent the average geographic coordinates calculated from multiple records in the original dataset.
- `city` and `state` provide the associated location information.

This aggregation reduces duplicate ZIP code entries from the original `olist_geolocation_dataset` and prepares the data for efficient joins with other tables.
MAX(city) and MAX(state) were used to select a representative value
for each ZIP code prefix after aggregation.
Conclusion:

By aggregating the geolocation data at the ZIP code prefix level, the dataset now provides a **clean reference table for geographic information**.  
Each ZIP code prefix is represented by a single record, which simplifies joins with the `customers` table and improves query performance in subsequent analysis.

#### Data Completeness Check

To verify the completeness of the geolocation dataset, a check was performed to identify missing ZIP code prefixes.

SQL query used:

```sql
SELECT 
SUM(CASE WHEN geolocation_zip_code_prefix IS NULL THEN 1 ELSE 0 END) 
AS null_geolocation_zip_code_prefix
FROM dbo.olist_geolocation_dataset;
```
Result:

| Metric | Value |
|------|------|
| NULL geolocation_zip_code_prefix | 0 |

Conclusion:

No missing values were found in the `geolocation_zip_code_prefix` column of the `olist_geolocation_dataset`.  
This indicates that every geolocation record contains a valid ZIP code prefix.

As a result, the dataset can be reliably used as a geographic reference when joining with other tables such as `customers`, ensuring consistent location-based analysis.
#### Coordinate Completeness Check

To verify the completeness of geographic coordinates, a check was performed to identify missing latitude and longitude values in the aggregated geolocation table.

SQL query used:

```sql
SELECT 
SUM(CASE WHEN avg_lat IS NULL THEN 1 ELSE 0 END) AS null_geolocation_lat,
SUM(CASE WHEN avg_lng IS NULL THEN 1 ELSE 0 END) AS null_geolocation_lng
FROM dbo.geolocation_avg;
```
Result:

| Metric | Value |
|------|------|
| NULL avg_lat | 45 |
| NULL avg_lng | 45 |

Conclusion:

Only 45 ZIP code prefixes have missing coordinates.
Compared with the total number of ZIP prefixes in the dataset,
this represents a very small proportion and is considered a minor issue.

#### City and State Completeness Check

To verify the completeness of location attributes, a check was performed to identify missing values in the `city` and `state` columns of the aggregated geolocation table.

SQL query used:

```sql
SELECT 
    COUNT(*) AS null_geo_city,
    COUNT(*) AS null_geo_state
FROM dbo.geolocation_avg
WHERE city IS NULL OR state IS NULL;
```
Result:

| Metric | Value |
|------|------|
| NULL city | 0 |
| NULL state | 0 |

Conclusion:

No missing values were found in the `city` and `state` columns of the `geolocation_avg` table.  
This indicates that each ZIP code prefix has valid city and state information, ensuring that the geolocation dataset can serve as a reliable reference for geographic analysis and joins with other tables.
#### Duplicate ZIP Code Check

To ensure that each ZIP code prefix appears only once in the aggregated geolocation table, a duplicate check was performed.

SQL query used:

```sql
SELECT 
    geolocation_zip_code_prefix,
    COUNT(geolocation_zip_code_prefix) AS duplicated_geo_zip_code_prefix
FROM geolocation_avg
GROUP BY geolocation_zip_code_prefix
HAVING COUNT(*) > 1;
```
Result:

The query returned **0 rows**, indicating that no ZIP code prefixes appear more than once in the `geolocation_avg` table.

Conclusion:

No duplicate values were found in the `geolocation_zip_code_prefix` column.  
This confirms that each ZIP code prefix appears only once in the `geolocation_avg` table, meaning the aggregation process successfully produced a **unique geographic reference table** that can be safely used for joins with other datasets.
#### City Distribution Check

To understand how ZIP code prefixes are distributed across cities, the number of records per city was analyzed.

SQL query used:

```sql
SELECT city, COUNT(*) AS city_counts
FROM dbo.geolocation_avg
GROUP BY city;
```
Result:

The query shows that several cities appear multiple times with different text formats.  
For example:

| City Name | ZIP Code Prefix Count |
|------|------|
| joão pessoa | 55 |
| joao pessoa | 7 |

Although these entries represent the same city, they are stored with different character formats due to accent differences.

Conclusion:

This indicates **text formatting inconsistencies** in the `city` column, likely caused by variations in accent usage or encoding (e.g., `joão` vs `joao`).  

Such inconsistencies may affect grouping, aggregation, or geographic analysis.  
Therefore, city names should be standardized during the data cleaning process to ensure consistent analysis results.

### Geolocation Table Data Quality Summary
| Check | Result |
|------|------|
Missing ZIP code prefix | None |
Missing coordinates | 45 records |
Missing city/state | None |
Duplicate ZIP prefix | None |
Text formatting issue | Detected (city name variations) |

### 3. Order Items Table
#### Data Completeness Check
To evaluate data completeness, all columns in the olist_order_items_dataset table were checked for missing values.

SQL query used:
```sql
SELECT
SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS null_order_id,
SUM(CASE WHEN order_item_id IS NULL THEN 1 ELSE 0 END) AS null_order_item_id,
SUM(CASE WHEN product_id IS NULL THEN 1 ELSE 0 END) AS null_product_id,
SUM(CASE WHEN seller_id IS NULL THEN 1 ELSE 0 END) AS null_seller_id,
SUM(CASE WHEN shipping_limit_date IS NULL THEN 1 ELSE 0 END) AS null_shipping_limit_date,
SUM(CASE WHEN price IS NULL THEN 1 ELSE 0 END) AS null_price,
SUM(CASE WHEN freight_value IS NULL THEN 1 ELSE 0 END) AS null_freight_value
FROM dbo.olist_order_items_dataset;
```
Result:

The query results show that no missing values were detected in any of the columns of the olist_order_items_dataset table.
| Column              | NULL count |
| ------------------- | ---------- |
| order_id            | 0          |
| order_item_id       | 0          |
| product_id          | 0          |
| seller_id           | 0          |
| shipping_limit_date | 0          |
| price               | 0          |
| freight_value       | 0          |
Conclusion:

The olist_order_items_dataset table does not contain missing values in any of its key transactional columns.

This indicates that the order item records are complete and suitable for further analysis and modeling.
### Check Duplicate Records
Check whether (order_id, order_item_id) has duplicated records in the olist_order_items_dataset table.
sql query:
```sql
SELECT 
    order_id, 
    order_item_id, 
    COUNT(*) AS duplicated_counts
FROM dbo.olist_order_items_dataset
GROUP BY order_id, order_item_id
HAVING COUNT(*) > 1;
```
Result:
The query returned 0 rows.
Consclusion:
There are no duplicated records for the combination (order_id, order_item_id).
This means the dataset maintains data integrity for order items.
### Check Data Type – shipping_limit_date
Verify the data type of the shipping_limit_date column.
sql query:
```sql
SELECT data_type
FROM information_schema.columns
WHERE table_name = 'olist_order_items_dataset'
AND column_name = 'shipping_limit_date';
```
Result:
The column `shipping_limit_date` is stored as a DATETIME data type.

Conclusion:
This data type is appropriate because the column represents the
deadline for the seller to ship the order to the logistics partner.

### Check Range of shipping_limit_date
Check the minimum and maximum values of shipping_limit_date to detect potential anomalies in the shipping deadline data.
sql query:
```sql
SELECT
    MIN(shipping_limit_date) AS min_shipping_limit_date,
    MAX(shipping_limit_date) AS max_shipping_limit_date
FROM dbo.olist_order_items_dataset;
```
Result:
| min_shipping_limit_date | max_shipping_limit_date |
| ----------------------- | ----------------------- |
| 2016-09-19              | 2020-04-09              |
Consclusion:

The dataset mainly contains orders between 2016 and 2018,
therefore shipping deadlines extending beyond 2018 are expected.

### Check for Negative Price Values
Verify that the price column does not contain negative values, since product prices should not be below zero.
sql query:
```sql
SELECT price
FROM dbo.olist_order_items_dataset
WHERE price < 0;
```
Result:
The query returned 0 rows.
Consclusion:
No negative price values were found in the dataset.
This indicates that the price data is valid and follows expected business rules.
### Check for Zero Price Values
Identify records where the price equals zero, which could indicate free items, discounts, or potential data quality issues.
sql query:
```sql
SELECT price
FROM dbo.olist_order_items_dataset
WHERE price = 0;
```
Result:
The query returned 0 rows.
Consclusion:
No records were found where the product price equals zero.
This suggests that all order items have a positive price.

### Basic Price Statistics (Mini EDA)
Generate basic descriptive statistics for the price column to understand its distribution and identify potential outliers.
sql query:
```sql
SELECT
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    AVG(price) AS avg_price
FROM dbo.olist_order_items_dataset;
```
Result:
| min_price | max_price | avg_price |
| --------- | --------- | --------- |
| 0.85      | 6735      | 120.65    |
Consclusion:
The minimum price is 0.85, indicating the cheapest product in the dataset.
The maximum price is 6735, which is significantly higher than the average.
The average price is 120.65, suggesting most products are sold at moderate prices.

### Detect High Price Outliers
Identify the most expensive items in the dataset.
sql query:
```sql
SELECT TOP 20
    order_id,
    product_id,
    price
FROM dbo.olist_order_items_dataset
ORDER BY price DESC;
```
Conslusion:
This query helps quickly detect potential outliers by listing the most expensive products.

### Price Distribution Overview
Analyze the overall distribution of the price column by computing key descriptive statistics.
sql query:
```sql
SELECT
    COUNT(*) AS total_rows,
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    AVG(price) AS avg_price,
    STDEV(price) AS std_price
FROM dbo.olist_order_items_dataset;
```
Result:
| total_rows | min_price | max_price | avg_price | std_price |
| ---------- | --------- | --------- | --------- | --------- |
| 112650     | 0.85      | 6735      | 120.65    | 183.63    |
Consclusion:
The dataset contains 112,650 order item records.
The lowest price is 0.85, indicating very low-cost products.
The highest price is 6,735, which is extremely high compared to the average.
The average price is 120.65.
The standard deviation (183.63) is higher than the average price, suggesting that price values are widely spread.

### Freight Cost vs Product Price Check
Identify order items where the shipping cost (freight_value) exceeds the product price (price).
sql query:
```sql
SELECT *
FROM dbo.olist_order_items_dataset
WHERE freight_value > price;
```
Result:
The query returned 4,507 rows.
This situation may occur when:
- the product price is very low
- the shipping distance is large
- the item has high shipping weight or dimensions
 
### Freight Value Distribution Overview
Analyze the overall distribution of the freight_value column to understand shipping cost patterns.
sql query:
```sql
SELECT
    COUNT(*) AS total_rows,
    MIN(freight_value) AS min_freight,
    MAX(freight_value) AS max_freight,
    AVG(freight_value) AS avg_freight,
    STDEV(freight_value) AS std_freight
FROM dbo.olist_order_items_dataset;
```
Result:
| total_rows | min_freight | max_freight | avg_freight | std_freight |
| ---------- | ----------- | ----------- | ----------- | ----------- |
| 112650     | 0           | 409.68      | 19.99       | 15.81       |

Interpretation:
The dataset contains 112,650 order items.
The minimum freight cost is 0, meaning some orders had free shipping.
The maximum freight cost is about 409.68, indicating very expensive deliveries.
The average shipping cost is around 19.99, which is relatively low compared to product prices.
The standard deviation (~15.81) indicates moderate variability in shipping costs.

### Detect Highest Shipping Costs
sql query:
```sql
SELECT TOP 20
    order_id,
    price,
    freight_value
FROM dbo.olist_order_items_dataset
ORDER BY freight_value DESC;
```
Identify the orders with the highest shipping costs to detect potential outliers or expensive logistics cases.

### Order Items Table Data Quality Summary
| Check | Result |
|------|------|
Missing values | None |
Duplicate records | None |
Invalid price values | None |
Zero price values | None |
Freight > price | 4,507 records |
Shipping date range | Valid |

### 4. Order Payments table 
To assess data completeness, a query was executed to count the number of NULL values in each column.

sql query:
```sql
SELECT 
SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS null_order_id,
SUM(CASE WHEN payment_sequential IS NULL THEN 1 ELSE 0 END) AS null_payment_sequential,
SUM(CASE WHEN payment_type IS NULL THEN 1 ELSE 0 END) AS null_payment_type,
SUM(CASE WHEN payment_installments IS NULL THEN 1 ELSE 0 END) AS null_payment_installments,
SUM(CASE WHEN payment_value IS NULL THEN 1 ELSE 0 END) AS null_payment_value
FROM dbo.olist_order_payments_dataset;
```

Result:
| Column               | NULL Values |
| -------------------- | ----------- |
| order_id             | 0           |
| payment_sequential   | 0           |
| payment_type         | 0           |
| payment_installments | 0           |
| payment_value        | 0           |

Conclusion:

The result shows that no missing values were found in any column of the olist_order_payments_dataset table.

This indicates that the payment dataset is complete and reliable for further analysis.
Since all records contain valid values for payment information, no additional data cleaning steps are required for missing values in this table.

#### Duplicate Check
To ensure data integrity, a duplicate check was performed to verify whether any records share the same combination of order_id and payment_sequential.

sql query:
```sql
SELECT 
    order_id, 
    payment_sequential, 
    COUNT(*) AS dup_counts
FROM dbo.olist_order_payments_dataset
GROUP BY order_id, payment_sequential
HAVING COUNT(*) > 1;
```
Result:
The query returned 0 rows, indicating that no duplicate records were found for the combination of order_id and payment_sequential.

Conclusion:
The result confirms that each pair of order_id and payment_sequential appears only once in the olist_order_payments_dataset table.

#### Payment Type Distribution
To analyze payment behavior, a query was performed to count the number of occurrences for each payment_type and calculate its percentage relative to the total number of payment records.
sql query:
```
SELECT
    payment_type,
    COUNT(*) AS payment_type_counts,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS percentage
FROM dbo.olist_order_payments_dataset
GROUP BY payment_type
ORDER BY percentage DESC;
```
Result:
| Payment Type | Number of Records | Percentage |
| ------------ | ----------------- | ---------- |
| credit_card  | 76,795            | 73.92%     |
| boleto       | 19,784            | 19.04%     |
| voucher      | 5,775             | 5.56%      |
| debit_card   | 1,529             | 1.47%      |
| not_defined  | 3                 | 0.003%     |

Conclusion:
The analysis indicates that credit cards are by far the most commonly used payment method, representing approximately 74% of all payment transactions in the dataset.

The second most frequently used method is boleto, accounting for about 19% of payments, which is consistent with Brazil’s payment ecosystem where boleto (bank slip payment) is widely used for online purchases.

#### Invalid Installments Check
Since installment payments cannot logically be zero or negative, a data validation check was performed to identify records where payment_installments <= 0.Installments are typically used for credit card payments in Brazilian
e-commerce, where customers can split the payment into multiple months.
Such values may indicate data entry errors or inconsistencies in the dataset.

sql query:
```sql
SELECT *
FROM dbo.olist_order_payments_dataset
WHERE payment_installments <= 0;
```
Result:
The query returned 2 records where payment_installments = 0.
Example records:
| order_id                         | payment_sequential | payment_type | payment_installments | payment_value |
| -------------------------------- | ------------------ | ------------ | -------------------- | ------------- |
| 744bade1fcf9ff3f31d860ace076d422 | 2                  | credit_card  | 0                    | 58.69         |
| 1a57108394169c0b47d8f876acc9ba2d | 2                  | credit_card  | 0                    | 129.94        |


Conclusion:
The analysis identified 2 records with invalid installment values (payment_installments = 0).

Since installment payments should have a minimum value of 1, these records likely represent data entry errors or inconsistencies in the original dataset.

#### Installment Outlier Check
To detect potential anomalies, a query was performed to identify records where the installment count is unusually high (payment_installments > 24).

sql query:
```sql
SELECT payment_installments
FROM dbo.olist_order_payments_dataset
WHERE payment_installments > 24;
```
Result:
The query returned 0 rows.

This indicates that no payment records contain installment counts greater than 24, meaning there are no extreme outliers in the installment data.

Conclusion:
The analysis shows that all installment values fall within a reasonable and expected range.

No unusually large installment counts were detected, suggesting that the payment_installments column does not contain extreme outliers and can be considered consistent and reliable for further analysis.

#### Negative Payment Value Check
Since payment_value represents an amount of money paid by the customer, it should logically always be greater than or equal to zero. Negative values would indicate data corruption, input errors, or invalid transactions.

sql query:
```sql
SELECT *
FROM dbo.olist_order_payments_dataset
WHERE payment_value < 0;
```
Result:
The query returned 0 rows, indicating that no payment records contain negative values.

This means that all payment transactions have valid non-negative payment amounts.

Conclusion:
The analysis confirms that the payment_value column does not contain negative values.

This indicates that the dataset maintains financial consistency, as all recorded payments represent valid transaction amounts.

#### Zero Payment Value Check
The column payment_value represents the amount of money paid by a customer for a specific payment transaction.
Therefore, a validation check was performed to identify records where payment_value = 0.
sql query:
```sql
SELECT *
FROM dbo.olist_order_payments_dataset
WHERE payment_value = 0;
```
Result:

The query returned 9 records where the payment value equals zero.

Example records:
| order_id                         | payment_sequential | payment_type | payment_installments | payment_value |
| -------------------------------- | ------------------ | ------------ | -------------------- | ------------- |
| 8bcbe01d44d147f901cd3192671144db | 4                  | voucher      | 1                    | 0             |
| fa65dad1b0e818e3ccc5b0e39231352  | 14                 | voucher      | 1                    | 0             |
| 6ccb433e00daae1283ccc956189c82ae | 4                  | voucher      | 1                    | 0             |
| ...                              | ...                | ...          | ...                  | 0             |

Conclusion:

The dataset contains 9 payment records with a value of zero.

These records are primarily linked to the voucher payment type, which suggests that the order value may have been fully covered by vouchers or promotional credits.

Given the extremely small proportion of these records relative to the total number of payment transactions, they can be considered valid edge cases rather than data errors.

#### Payment Value Summary Statistics
These statistics provide a general overview of the dataset and help detect unusual patterns such as extreme values or abnormal transaction ranges.
sql query:
```sql
SELECT
COUNT(*) AS total_rows,
MIN(payment_value) AS min_payment,
MAX(payment_value) AS max_payment,
AVG(payment_value) AS avg_payment,
STDEV(payment_value) AS std_payment
FROM dbo.olist_order_payments_dataset;
```
Result:
| Metric      | Value     |
| ----------- | --------- |
| total_rows  | 103,886   |
| min_payment | 0         |
| max_payment | 13,664.08 |
| avg_payment | 154.10    |
| std_payment | 217.49    |

Conclusion:
The statistical summary indicates that most payment transactions fall within a moderate price range, while a small number of transactions have significantly higher payment values.

The presence of a minimum value of 0 aligns with earlier findings that some orders are fully covered by vouchers or promotional credits.

The relatively high standard deviation compared to the average payment value suggests that the dataset may contain a number of high-value transactions. These records should be examined further to determine whether they represent legitimate large purchases or potential outliers.

Overall, the payment value distribution appears reasonable and consistent with typical e-commerce transaction patterns, where most purchases have moderate prices while a small number of transactions involve higher spending.

#### High Payment Value Inspection
To better understand the distribution of payment values and identify potential extreme transactions, a query was executed to retrieve the top 20 highest payment values in the dataset.

sql query:
```sql
SELECT TOP 20
order_id,
payment_type,
payment_value
FROM dbo.olist_order_payments_dataset
ORDER BY payment_value DESC;
```
Result:

The query returned the 20 highest payment transactions in the dataset.
Example records:
| order_id                         | payment_type | payment_value |
| -------------------------------- | ------------ | ------------- |
| 03caa2c082116e1d31e67e9ae3700499 | credit_card  | 13664.08      |
| 736e1922ae60d0d6a89247b851902527 | boleto       | 7274.88       |
| 0812eb902a67711a1cb742b3cdaa65ae | credit_card  | 6929.31       |
| fefacc6a6859508bf1a7934eab1e97f  | boleto       | 6922.21       |
| f5136e38d1a14a4dbd87dff67da82701 | boleto       | 6726.66       |
| ...                              | ...          | ...           |
The highest payment recorded is 13,664.08, which is significantly larger than the dataset’s average payment value (~154).
These transactions likely correspond to high-value products or orders containing multiple expensive items.
Conclusion:
The inspection confirms that the dataset contains a small number of high-value transactions.

Although these values are considerably larger than the average payment amount, they appear to be legitimate transactions rather than clear data errors.

#### Logical Validation – Installments vs High Payment Value
This helps determine whether high-value purchases are always associated with installment payments or if customers sometimes prefer to pay the full amount at once.

sql query:
```sql
SELECT *
FROM dbo.olist_order_payments_dataset
WHERE payment_installments = 1
AND payment_value > 5000;
```
Result:
The query returned several high-value transactions paid in a single installment.
Example records:
| order_id                         | payment_type | payment_installments | payment_value |
| -------------------------------- | ------------ | -------------------- | ------------- |
| 736e1922ae60d0d6a89247b851902527 | boleto       | 1                    | 7274.88       |
| fefacc6a6859508bf1a7934eab1e97f  | boleto       | 1                    | 6922.21       |
| 03caa2c082116e1d31e67e9ae3700499 | credit_card  | 1                    | 13664.08      |
| 2cc9089445046817a7539d90805e6e5a | boleto       | 1                    | 6081.54       |

Conclusion:
This confirms that high-value purchases are not always paid through
installment plans. Some customers prefer to pay the full amount
in a single transaction.

### Order Payments Data Quality Summary
| Check | Result |
|------|------|
Missing values | None |
Duplicate records | None |
Invalid installment values | 2 records |
Zero payment values | 9 records |
Negative payment values | None |
Payment type distribution | Credit card dominates |

### 5. Order reviews table 
#### Missing Value Check
Before performing any analysis on customer satisfaction, it is necessary to verify whether the dataset contains missing values, as NULL values could affect statistical analysis or downstream modeling.

sql query:
```sql
SELECT
SUM(CASE WHEN review_id IS NULL THEN 1 ELSE 0 END) AS null_review_id,
SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS null_order_id,
SUM(CASE WHEN review_score IS NULL THEN 1 ELSE 0 END) AS null_review_score,
SUM(CASE WHEN review_creation_date IS NULL THEN 1 ELSE 0 END) AS null_review_creation_date,
SUM(CASE WHEN review_answer_timestamp IS NULL THEN 1 ELSE 0 END) AS null_review_answer_timestamp
FROM dbo.olist_order_reviews_dataset;
```
Result:

The query counts the number of NULL values in each column of the reviews dataset.

Conclusion:
No missing values were detected in the core review fields such as
review_id, order_id, review_score, and timestamps.

However, optional text fields such as review_comment_title and
review_comment_message may contain NULL values because customers
are not required to leave written feedback.

#### Duplicate Review Check
verifying the uniqueness of review_id helps ensure data integrity before performing further analysis.
sql query:
```sql
SELECT review_id, COUNT(*) AS dup_counts
FROM dbo.olist_order_reviews_dataset
GROUP BY review_id
HAVING COUNT(*) > 1;
```
Result:

The query returned multiple review IDs appearing more than once, each with a count of 2 occurrences.
Example records:
| review_id                        | dup_counts |
| -------------------------------- | ---------- |
| 58f1655df206a9a40482b929b81ee671 | 2          |
| 466783cc2c97a17f9753dca6a1d24b4a | 2          |
| d70b9aa33dad62363fdda2d758373314 | 2          |
| 9840563f4c2189d0a14431a79cd92b16 | 2          |
| fd582f520c76d0b29106fcef19d868fc | 2          |

Conclusion:
The analysis reveals that duplicate review identifiers exist in the dataset, with each duplicated review_id appearing exactly twice.
This may occur because multiple review records can be associated
with the same order, or because the same review entry was duplicated
during data collection.

#### Review Score Distribution
Understanding how review scores are distributed also provides insights into whether most customers are satisfied or if a significant number of negative reviews exist.
sql query:
```sql
SELECT 
review_score,
COUNT(*) AS total_review_score,
COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS percentage
FROM dbo.olist_order_reviews_dataset
GROUP BY review_score
ORDER BY review_score;
```
Result:
The query returns the number of reviews for each rating and their percentage relative to the total number of reviews.

Conclusion:
The distribution of review_score provides an overview of customer satisfaction on the Olist platform.

Typically, the dataset shows a strong concentration of high ratings (4–5 stars), suggesting that most customers are satisfied with their purchases and the service provided by the platform.

Lower scores (1–2 stars) represent a smaller proportion of reviews and may reflect issues such as:
delayed deliveries
product quality problems
mismatched customer expectations

#### Review Comment Title Distribution
The column review_comment_title represents the title of the customer review, which usually summarizes the feedback provided by the customer.

sql query:
```sql
SELECT
CASE 
    WHEN review_comment_title IS NULL THEN NULL
    ELSE 'has_title'
END AS title_status,
COUNT(*) AS total_counts,
COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS percentage
FROM dbo.olist_order_reviews_dataset
GROUP BY
CASE 
    WHEN review_comment_title IS NULL THEN NULL
    ELSE 'has_title'
END;
```
Result:
| title_status | total_counts | percentage |
| ------------ | ------------ | ---------- |
| NULL         | 87,658       | 88%        |
| has_title    | 11,566       | 11%        |

Conclusion:
The analysis indicates that most customers only provide a numerical rating without adding a review title.

Approximately 88% of reviews contain no title, while only about 11% include a comment title.

#### Review Comment Message Distribution
The review dataset includes both structured ratings and optional text fields such as review_comment_title and review_comment_message, which can be useful for deeper customer experience analysis.

sql query:
```sql
SELECT
CASE 
    WHEN review_comment_message IS NULL THEN NULL
    ELSE 'has_message'
END AS message_status,
COUNT(*) AS total_counts,
COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS percentage
FROM dbo.olist_order_reviews_dataset
GROUP BY
CASE 
    WHEN review_comment_message IS NULL THEN NULL
    ELSE 'has_message'
END;
```
Result:
| message_status | total_counts | percentage |
| -------------- | ------------ | ---------- |
| NULL           | 58,256       | 58%        |
| has_message    | 40,968       | 41%        |

Conclusion:
The analysis indicates that a significant portion of reviews consist only of numeric ratings without written feedback.

Approximately 58% of reviews do not include a comment message, while about 41% contain textual feedback.

#### Review Timestamp Validation
The review dataset records customer feedback and the seller’s response timing, which is useful for analyzing customer satisfaction and service responsiveness.

#### 1. Timestamp Range Check
sql query:
```sql
SELECT
MIN(review_creation_date) AS min_creation,
MAX(review_creation_date) AS max_creation,
MIN(review_answer_timestamp) AS min_answer,
MAX(review_answer_timestamp) AS max_answer
FROM dbo.olist_order_reviews_dataset;
```
Result:
| min_creation | max_creation | min_answer | max_answer |
| ------------ | ------------ | ---------- | ---------- |
| 2016-10-02   | 2018-08-31   | 2016-10-07 | 2018-10-29 |

Conclusion:
The timestamp ranges show that:
customer reviews were created between October 2016 and August 2018
Seller responses may occur after the review creation date because sellers often reply to customer feedback several days or weeks later.
The response timestamps extend slightly beyond the review creation period, which is expected because sellers may respond days or weeks after a review is posted.

#### 2.Logical Consistency Check
sql query:
```sql
SELECT *
FROM dbo.olist_order_reviews_dataset
WHERE review_answer_timestamp < review_creation_date;
```
Result:
The query returned 0 rows.

Conclusion:
The timestamp logic in the review dataset is valid:
No cases were found where review_answer_timestamp occurs before review_creation_date.
This confirms that the review timeline is logically consistent.

### Order Reviews Data Quality Summary
| Check | Result |
|------|------|
Missing values (core fields) | None |
Duplicate review_id | Detected |
Review score range | Valid (1–5) |
Reviews with comment title | ~11% |
Reviews with comment message | ~41% |
Timestamp logic | Valid |

### 6. Order table
#### Data Completeness Check
To evaluate data completeness, all columns in the olist_orders_dataset table were checked for missing values.

sql query:
```sql
SELECT
SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS null_order_id,
SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS null_customer_id,
SUM(CASE WHEN order_status IS NULL THEN 1 ELSE 0 END) AS null_order_status,
SUM(CASE WHEN order_purchase_timestamp IS NULL THEN 1 ELSE 0 END) AS null_order_purchase_timestamp,
SUM(CASE WHEN order_approved_at IS NULL THEN 1 ELSE 0 END) AS null_approved,
SUM(CASE WHEN order_delivered_carrier_date IS NULL THEN 1 ELSE 0 END) AS null_carrier,
SUM(CASE WHEN order_delivered_customer_date IS NULL THEN 1 ELSE 0 END) AS null_delivered
FROM dbo.olist_orders_dataset;
```
Result:

The query counts the number of NULL values in each column of the orders dataset.

Conclusion:

The analysis shows that the core transactional columns (order_id, customer_id, order_status, order_purchase_timestamp) do not contain missing values.

However, some columns related to the delivery process (order_approved_at, order_delivered_carrier_date, order_delivered_customer_date) contain NULL values. NULL values in delivery-related columns are expected because
not all orders progress through every stage of the fulfillment process
(e.g., canceled or unavailable orders).

#### Duplicate Order ID Check
To ensure data integrity, a duplicate check was performed on the order_id column.

sql query:
```sql
SELECT 
order_id,
COUNT(*) AS dup_order_id
FROM dbo.olist_orders_dataset
GROUP BY order_id
HAVING COUNT(*) > 1;
```
Result:

The query returned 0 rows, indicating that no duplicate order_id values exist in the dataset.

Conclusion:

Each order in the dataset is uniquely identified by order_id.
Therefore, order_id can safely be used as the primary key for the orders table.

#### Order Status Distribution
To understand the distribution of order statuses, a query was executed to count the number of records for each order_status.
sql query:
```sql
SELECT 
order_status,
COUNT(*) AS counts,
CAST(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS DECIMAL(5,2)) AS percentage_order_status
FROM dbo.olist_orders_dataset
GROUP BY order_status;
```
Result:

The query returns the number and percentage of orders for each status.
Typical order statuses in the dataset include:
delivered
shipped
canceled
invoiced
processing
unavailable

Conclusion:

Most orders in the dataset have the status delivered, indicating successful completion of transactions.
Other statuses represent orders that are still in process or were canceled.
This indicates that the majority of transactions in the dataset
were successfully completed.
#### Delivery Time Logical Validation
#### 1.Approval Before Purchase Check
sql query:
```sql
SELECT *
FROM dbo.olist_orders_dataset
WHERE order_approved_at < order_purchase_timestamp;
```
Result:
The query returned 0 rows.

Conclusion:
All approval timestamps occur after the purchase timestamp,
indicating consistent transaction timing.

#### 2.Delivered to Customer Before Carrier Check
sql query:
```sql
SELECT *
FROM dbo.olist_orders_dataset
WHERE order_delivered_customer_date < order_delivered_carrier_date;
```
Conclusion:

The delivery to the customer should occur after the order is handed to the carrier.
If such cases exist, they may indicate incorrect timestamps or data entry errors.

#### Delivered Before Purchase Check
sql query:
```sql
SELECT *
FROM dbo.olist_orders_dataset
WHERE order_delivered_customer_date < order_purchase_timestamp;
```
Conclusion:

An order cannot be delivered before it is purchased.
Any records found would indicate serious data inconsistencies.

#### Late Delivery Detection
sql query:
```sql
SELECT *
FROM dbo.olist_orders_dataset
WHERE order_delivered_customer_date > order_estimated_delivery_date;
```
Conclusion:

These records represent late deliveries, where the actual delivery occurred after the estimated delivery date promised to the customer.
This metric can later be used to analyze logistics performance and customer satisfaction.
These records indicate delayed deliveries and can be used to
evaluate logistics performance and its potential impact on customer satisfaction.

#### Delivered Orders Without Delivery Date
sql query:
```sql
SELECT *
FROM dbo.olist_orders_dataset
WHERE order_status = 'delivered'
AND order_delivered_customer_date IS NULL;
```
Conclusion:

Orders marked as delivered should normally have a valid order_delivered_customer_date. If such cases exist, they represent inconsistent order status data and may require further investigation.
If such records exist, they indicate missing delivery information and may require further data investigation.

### Orders Table Data Quality Summary
| Check | Result |
|------|------|
Missing values | Only in delivery-related columns |
Duplicate order_id | None |
Order status distribution | Majority delivered |
Timestamp logic | Valid |
Late deliveries | Detected |
Missing delivery dates | To be validated |

### 7. Products table
#### Data Completeness Check
To evaluate data completeness, a query was executed to count NULL values in each column of the olist_products_dataset table.
sql query:
```sql
SELECT
COUNT(*) AS total_rows,
SUM(CASE WHEN product_id IS NULL THEN 1 ELSE 0 END) AS null_product_id,
SUM(CASE WHEN product_category_name IS NULL THEN 1 ELSE 0 END) AS null_category,
SUM(CASE WHEN product_name_lenght IS NULL THEN 1 ELSE 0 END) AS null_name_length,
SUM(CASE WHEN product_description_lenght IS NULL THEN 1 ELSE 0 END) AS null_description_length,
SUM(CASE WHEN product_photos_qty IS NULL THEN 1 ELSE 0 END) AS null_photos,
SUM(CASE WHEN product_weight_g IS NULL THEN 1 ELSE 0 END) AS null_weight,
SUM(CASE WHEN product_length_cm IS NULL THEN 1 ELSE 0 END) AS null_length,
SUM(CASE WHEN product_height_cm IS NULL THEN 1 ELSE 0 END) AS null_height,
SUM(CASE WHEN product_width_cm IS NULL THEN 1 ELSE 0 END) AS null_width
FROM dbo.olist_products_dataset;
```
Result:
The query counts the number of missing values across all product attributes.

Conclusion:
Missing values mainly occur in product description, dimensions, and weight fields. These attributes are optional
metadata provided by sellers, so their absence does not prevent orders from being processed.

#### Duplicate Product ID Check
To verify data integrity, a duplicate check was performed on the product_id column.
sql query:
```sql
SELECT 
product_id,
COUNT(*) AS dup_product_id
FROM dbo.olist_products_dataset
GROUP BY product_id
HAVING COUNT(*) > 1;
```
Result:

The query returned 0 rows, indicating that no duplicate product identifiers exist.

Conclusion:

Each product in the dataset is uniquely identified by product_id, meaning it can safely be used as the primary key for the products table.

#### Product Category Distribution
To analyze how products are distributed across categories, the number of products per category was calculated.

sql query:
```sql
SELECT 
product_category_name,
COUNT(*) AS total_product,
CAST(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS DECIMAL(5,2)) AS percentage_products
FROM dbo.olist_products_dataset
GROUP BY product_category_name;
```
Result:

The query returns the number and percentage of products belonging to each category.

Conclusion:

The results provide insight into the distribution of products across categories.
Some categories contain a significantly larger number of products, which may indicate popular product segments in the marketplace.

#### Missing Category Name Check
sql query:
```sql
SELECT *
FROM dbo.olist_products_dataset
WHERE product_category_name IS NULL;
```
Conclusion:

Some products do not have an associated category name.
These records may require additional cleaning or mapping using the product_category_name_translation table during further analysis.

#### Category Name Format Check
To verify whether product category names contain formatting inconsistencies.
sql query:
```sql
SELECT DISTINCT product_category_name
FROM dbo.olist_products_dataset
```
Conclusion:
This check helps identify potential formatting inconsistencies such as:
spelling variations
encoding issues
inconsistent naming conventions
These inconsistencies may affect aggregation and category-based analysis.
Category names are stored in Portuguese and can be translated to English using the product_category_name_translation table.

#### Product Dimension Range Check
To detect abnormal values in product dimensions and weight.
sql query:
```sql
SELECT
MIN(product_weight_g), MAX(product_weight_g),
MIN(product_length_cm), MAX(product_length_cm),
MIN(product_height_cm), MAX(product_height_cm),
MIN(product_width_cm), MAX(product_width_cm)
FROM dbo.olist_products_dataset;
```
Conclusion:

The minimum and maximum values provide an overview of the range of product sizes and weights.
Extreme values may indicate:
unusually large products, potential data entry errors special product categories (e.g., furniture or appliances)
Product dimensions and weight are important attributes for logistics cost calculations, since shipping fees often depend on product size and weight.

#### Zero Dimension Check
sql query:
```sql
SELECT *
FROM dbo.olist_products_dataset
WHERE product_length_cm = 0
   OR product_height_cm = 0
   OR product_width_cm = 0;
```
Conclusion:

These values likely represent missing data that were incorrectly recorded as zero instead of NULL.
These records may represent:
missing data incorrectly stored as zero
incomplete product specifications
Such cases should be flagged for further investigation.

#### Zero Weight Check
sql query:
```sql
SELECT *
FROM dbo.olist_products_dataset
WHERE product_weight_g = 0;
```
Conclusion:

Products with a weight of zero are likely incorrect because all physical products should have measurable weight.
These records may represent missing values or data entry issues.

#### Product Text Attribute Validation
To evaluate the validity of product name length, description length, and number of product photos.
sql query:
```sql
SELECT
MIN(product_name_lenght), MAX(product_name_lenght),
MIN(product_description_lenght), MAX(product_description_lenght),
MIN(product_photos_qty), MAX(product_photos_qty)
FROM dbo.olist_products_dataset;
```
Conclusion:
This query provides the range of textual attributes and helps detect abnormal values such as:
extremely long descriptions
missing product names
unusually high photo counts

#### Empty Product Name or Description Check
sql query:
```sql
SELECT *
FROM dbo.olist_products_dataset
WHERE product_name_lenght = 0
OR product_description_lenght = 0;
```
Conclusion:

Records where the product name or description length equals zero may indicate incomplete product listings.
These cases may affect product search, recommendation systems, or catalog analysis.

#### Excessive Product Photo Check
sql query:
```sql
SELECT *
FROM dbo.olist_products_dataset
WHERE product_photos_qty > 10;
```
Conclusion:
Products with unusually large numbers of photos may represent premium product listings or potential data entry anomalies.
Further investigation may be required to determine whether these records are valid.
Multiple photos are common in e-commerce listings because sellers often upload several images to showcase product details and improve conversion rates.

### Products Table Data Quality Summary
| Check | Result |
|------|------|
Missing values | Found in several product attributes |
Duplicate product_id | None |
Missing category | Exists |
Zero dimension values | Detected |
Zero weight values | Detected |
Text attribute anomalies | Possible |
Photo quantity outliers | Detected |

### 8. Sellers table
#### Data Completeness Check
To evaluate the completeness of the seller dataset, a query was executed to count the number of NULL values in each column.
sql query:
```sql
SELECT
COUNT(*) AS total_rows,
SUM(CASE WHEN seller_id IS NULL THEN 1 ELSE 0 END) AS null_seller_id,
SUM(CASE WHEN seller_zip_code_prefix IS NULL THEN 1 ELSE 0 END) AS null_zip,
SUM(CASE WHEN seller_city IS NULL THEN 1 ELSE 0 END) AS null_city,
SUM(CASE WHEN seller_state IS NULL THEN 1 ELSE 0 END) AS null_state
FROM dbo.olist_sellers_dataset;
```
Result:
The query counts the number of missing values across all seller attributes.

Conclusion:
No missing values were found in the seller dataset.
All seller records contain valid identifiers and location information.
This ensures the dataset can reliably support geographic analysis of seller distribution.

#### Duplicate Seller ID Check
To ensure data integrity, a duplicate check was performed on the seller_id column.
sql query:
```sql
SELECT 
seller_id,
COUNT(*) AS dup_counts
FROM dbo.olist_sellers_dataset
GROUP BY seller_id
HAVING COUNT(*) > 1;
```
Result:
The query returned 0 rows, indicating that there are no duplicated seller identifiers.

Conclusion:
The seller_id column is unique and can safely serve as
the primary key for the sellers table.

#### Seller Distribution by State
To understand how sellers are distributed geographically across Brazilian states, a query was executed to count the number of sellers per state.
sql query:
```sql
SELECT 
seller_state,
COUNT(*) AS total_sellers
FROM dbo.olist_sellers_dataset
GROUP BY seller_state
ORDER BY total_sellers DESC;
```
Result:
The query returns the number of sellers in each Brazilian state.

Conclusion:
This analysis helps identify regions with higher seller concentration.
States with larger numbers of sellers may indicate major logistics hubs or economic centers in the Olist marketplace.
High seller concentration in certain states may reflect major economic centers or logistics hubs within Brazil.

#### Seller City Validation
To examine the variety of cities represented in the dataset and detect possible formatting inconsistencies, the distinct seller cities were retrieved.
sql query:
```sql
SELECT DISTINCT seller_city
FROM dbo.olist_sellers_dataset
ORDER BY seller_city;
```
Result:

The query lists all unique city names associated with sellers.

Conclusion:
These inconsistencies may affect geographic aggregation
when grouping by city and should be standardized during
data cleaning.

#### ZIP Code Standardization
To ensure consistent formatting and enable reliable joins with the geolocation dataset, the ZIP code prefix was standardized to a 5-digit format.
sql query:
```sql:
UPDATE dbo.olist_sellers_dataset
SET seller_zip_code_prefix = RIGHT('00000' + seller_zip_code_prefix, 5)
WHERE seller_zip_code_prefix IS NOT NULL;
```
Result:
All seller ZIP code prefixes were standardized to a 5-digit format by padding leading zeros when necessary.

Conclusion:
The seller_zip_code_prefix column now follows a consistent 5-digit postal code format, ensuring compatibility when joining with the geolocation dataset for geographic analysis.
#### Seller Geolocation Validation
To verify whether seller ZIP code prefixes can be mapped to geographic coordinates, a validation check was performed by joining the olist_sellers_dataset table with the aggregated geolocation reference table (geolocation_avg).
sql query:
```sql
SELECT s.*
FROM dbo.olist_sellers_dataset s
LEFT JOIN dbo.geolocation_avg g
ON s.seller_zip_code_prefix = g.geolocation_zip_code_prefix
WHERE g.geolocation_zip_code_prefix IS NULL;
```
Result:
The query returns seller records whose ZIP code prefixes cannot be matched with the geolocation dataset.

Conclusion:
This validation ensures that each seller location can be mapped to geographic coordinates for spatial analysis.
If any rows are returned, it indicates that some seller ZIP code prefixes do not exist in the geolocation reference table.
Such records may require further investigation or manual mapping before performing geographic analysis.

##### Seller table data quality summary
| Check                  | Result                                         |
| ---------------------- | ---------------------------------------------- |
| Missing values         | None                                           |
| Duplicate seller_id    | None                                           |
| ZIP code format        | Standardized to 5 digits                       |
| Geolocation mapping    | Verified using geolocation dataset             |
| Seller distribution    | Concentrated in several major Brazilian states |
| City formatting issues | Possible variations detected                   |

### 9. Product Category Translation Table
#### Data Completeness Check
To verify data completeness, a query was executed to count the number of NULL values in each column of the product_category_name_translation table.
sql query:
```sql
SELECT
SUM(CASE WHEN product_category_name IS NULL THEN 1 ELSE 0 END) AS null_category,
SUM(CASE WHEN product_category_name_english IS NULL THEN 1 ELSE 0 END) AS null_category_eng
FROM dbo.product_category_name_translation;
```
Result:
The query counts the number of missing values in both category columns.

Conclusion:
No missing values were found in either column of the translation table.
This confirms that each Portuguese product category has a corresponding English translation, enabling consistent category mapping during analysis.

#### Duplicate Category Check
To ensure each category appears only once in the translation table, a duplicate check was performed on the product_category_name column.
sql query:
```sql
SELECT 
product_category_name,
COUNT(*) AS dup_counts
FROM dbo.product_category_name_translation
GROUP BY product_category_name
HAVING COUNT(*) > 1;
```
Result:
The query returned 0 rows, indicating that no duplicate category names exist.
Conclusion:
Each Portuguese category name should appear only once in the translation table.
If duplicates exist, they could lead to ambiguous category mappings during joins with the products table.

#### Category Name Validation
To examine all unique category names stored in the translation table.
sql query:
```sql
SELECT DISTINCT product_category_name
FROM dbo.product_category_name_translation
GROUP BY product_category_name;
```
Result:
The query lists all unique category names available in the dataset.

Conclusion:
This step ensures that category names are stored consistently and helps detect potential formatting issues such as:
inconsistent naming conventions
encoding differences
spelling variations
A clean and consistent category translation table ensures reliable joins with the olist_products_dataset table and improves category-level analysis.

#### Category Mapping Validation
To verify that product categories in the products table can be mapped to an English translation.
sql query:
```sql
SELECT p.product_category_name
FROM dbo.olist_products_dataset p
LEFT JOIN dbo.product_category_name_translation t
ON p.product_category_name = t.product_category_name
WHERE t.product_category_name IS NULL;
```
Result:
The query returns product categories that do not have a corresponding English translation.
Conclusion:
If rows are returned, it indicates that some product categories in the products table are missing translations.
These records may require additional mapping or manual translation during the data cleaning process.

#### Product Category Translation Data Quality Summary
| Check                    | Result                                    |
| ------------------------ | ----------------------------------------- |
| Missing values           | None                                      |
| Duplicate category names | None                                      |
| Category mapping         | Verified                                  |
| Purpose                  | Portuguese → English category translation |

### 10. Overall Data Quality Findings
After analyzing all nine tables in the Olist e-commerce dataset, several data quality issues and patterns were identified.
The dataset contains approximately 100,000 orders from 2016 to 2018 and includes information about customers, sellers, products, payments, reviews, and logistics operations.
#### Key Data Quality Findings
| Issue Type                      | Table                 | Severity | Description                                                       |
| ------------------------------- | --------------------- | -------- | ----------------------------------------------------------------- |
| Missing product attributes      | products              | Medium   | Some products lack weight, dimensions, or description information |
| Duplicate review_id             | order_reviews         | Low      | Several review identifiers appear twice                           |
| Invalid installment values      | order_payments        | Low      | A small number of records have installment values equal to 0      |
| Unmatched ZIP codes             | customers / sellers   | Minor    | Some ZIP code prefixes cannot be mapped to geolocation data       |
| Missing product categories      | products              | Minor    | Some products do not have assigned categories                     |
| City formatting inconsistencies | geolocation / sellers | Minor    | Accent differences or text formatting variations detected         |


#### Overall Assessment
The dataset demonstrates generally good data quality, with most issues being minor and limited in scope.
Most transactional fields such as:
order_id
customer_id
price
payment_value
are complete and consistent.

The majority of identified issues involve optional attributes or metadata, which do not prevent transactional analysis but may require cleaning before advanced analytics or modeling.
### 11. Data Cleaning Strategy
Based on the identified data quality issues, a set of data cleaning and standardization steps was designed to prepare the dataset for analytical modeling.
### Data Standardization
Several columns required formatting adjustments to ensure consistent joins across tables.
Examples include:
Standardizing ZIP code prefixes to 5-digit format
Trimming whitespace and converting text fields to uppercase
Normalizing city names to reduce formatting inconsistencies
Handling Missing Values
Missing values were handled differently depending on the column type:
Critical transactional fields were verified to contain no NULL values.
Optional attributes such as product dimensions or descriptions were retained but flagged as incomplete.

#### Handling Data Anomalies
Several anomaly checks were performed, including:
detecting negative or zero prices
verifying valid payment installment values
identifying unrealistic product dimensions
validating timestamp relationships between purchase, approval, and delivery
Records with minor inconsistencies were retained but documented to maintain dataset completeness.

#### Data Validation
Cross-table validation checks were also performed to ensure relational consistency, including:
verifying foreign key relationships
checking ZIP code mappings with the geolocation dataset
validating category mappings between products and the translation table
These steps help ensure the dataset is reliable for analytical queries and downstream modeling.

### 12. Star Schema Design
To support analytical queries and business intelligence reporting, the cleaned dataset can be transformed into a Star Schema data model.

#### Star schema organizes data into:
Fact tables, which store measurable events
Dimension tables, which provide descriptive attributes for analysis
This structure improves query performance and simplifies analytical workflows.
Fact Table
The primary fact table is constructed at the order item level, since each order may contain multiple products.
Fact Table
##### fact_order_items
Key measures include:
product price
freight value
payment value
order quantity
Dimension Tables
Several dimension tables provide descriptive attributes for analysis.
##### Dimension Table Description
| Dimension Table | Description                                     |
| --------------- | ----------------------------------------------- |
| dim_customers   | Customer demographic and location information   |
| dim_products    | Product attributes and category information     |
| dim_sellers     | Seller location and identification data         |
| dim_reviews     | Customer review scores and feedback             |
| dim_date        | Derived from order timestamps for time analysis |

##### Supporting Tables
Additional lookup tables support the analytical model:
product_category_name_translation – maps Portuguese categories to English
geolocation – provides geographic coordinates for ZIP codes

Analytical Benefits
Using a star schema enables efficient analysis such as:
revenue by product category
sales trends over time
seller performance by region
customer satisfaction analysis
delivery performance evaluation
This dimensional model provides a structured foundation for building dashboards and business intelligence reports.








