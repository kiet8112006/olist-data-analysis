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
### Geolocation Data Aggregation

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

