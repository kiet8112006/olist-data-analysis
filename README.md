# olist-data-analysis
SQL Data Quality Analysis for Olist E-commerce Dataset
## Dataset Overview

The dataset contains information about customers, orders, order items,
payments, reviews, products, sellers, geolocation, and product category
translations.

Total tables analyzed: 9

---

## Data Quality Dimensions

- Data completeness
- Data consistency
- Data uniqueness
- Data validity

---
## Data Model

The Olist dataset is a relational e-commerce database composed of multiple interconnected tables.  
These tables represent customers, orders, products, sellers, payments, reviews, and geographical information.

Understanding the relationships between tables is essential for accurate data analysis and proper data quality validation.

### Main Transaction Flow

The core transaction flow of the dataset follows this structure:

customers → orders → order_items → products

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
select count(*) as null_customer_id 
from dbo.olist_customers_dataset
where customer_id is null;

select count(*) as null_customer_unique_id 
from dbo.olist_customers_dataset
where customer_unique_id is null;

select count(*) as null_customer_zip_code_prefix 
from dbo.olist_customers_dataset
where customer_zip_code_prefix is null;

select count(*) as null_customer_city 
from dbo.olist_customers_dataset
where customer_city is null;

select count(*) as null_customer_state 
from dbo.olist_customers_dataset
where customer_state is null;
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

The `customer_id` column is unique across the dataset and can safely be used as the **primary key** for identifying individual customer order records.

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
#### ZIP Code Standardization

The column `customer_zip_code_prefix` represents the customer's postal code prefix.  
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

SQL queries used:

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

Some customer ZIP codes do not exist in the geolocation reference table.  
These records may require additional investigation or handling during geographic analysis.
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

Conclusion:

By aggregating the geolocation data at the ZIP code prefix level, the dataset now provides a **clean reference table for geographic information**.  
Each ZIP code prefix is represented by a single record, which simplifies joins with the `customers` table and improves query performance in subsequent analysis.

#### Data Completeness Check

To verify the completeness of the geolocation dataset, a check was performed to identify missing ZIP code prefixes.

SQL query used:

```sql
SELECT COUNT(*) AS null_geolocation_zip_code_prefix
FROM dbo.olist_geolocation_dataset
WHERE geolocation_zip_code_prefix IS NULL;
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

A total of **45 ZIP code prefixes** in the `geolocation_avg` table have missing geographic coordinates (`avg_lat` and `avg_lng`).  
These records indicate locations where latitude and longitude information is unavailable after aggregating the original geolocation dataset.

However, compared with the total number of ZIP code prefixes in the dataset, this issue affects only a **very small portion of the data** and is considered a **minor data quality issue**.

These records may be excluded or handled separately during geographic analysis if precise location coordinates are required.
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
### 3. Order Items Table
Data Completeness Check

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
The shipping deadline dates range from September 2016 to April 2020, which falls within the expected time frame of the Olist dataset.
No abnormal values were detected.

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
    STD(price) AS std_price
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
Interpretation:
There are 4,507 order items where the shipping cost is higher than the product price.
 
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

### 4. Order payments table 
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

Consclusion:
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

Consclusion:
The analysis indicates that credit cards are by far the most commonly used payment method, representing approximately 74% of all payment transactions in the dataset.

The second most frequently used method is boleto, accounting for about 19% of payments, which is consistent with Brazil’s payment ecosystem where boleto (bank slip payment) is widely used for online purchases.

#### Invalid Installments Check
Since installment payments cannot logically be zero or negative, a data validation check was performed to identify records where payment_installments <= 0.
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


Consclusion:
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

Consclusion:
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
The analysis shows that large payment values can occur even when payment_installments = 1.

### 5. order reviews table 

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
The results indicate that no missing values were found in the main columns of the olist_order_reviews_dataset table.

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

Consclusion:
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
seller responses occurred between October 2016 and October 2018
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











