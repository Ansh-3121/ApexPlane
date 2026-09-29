# Sales Data Analysis using MySQL

**ApexPlanet Software Pvt. Ltd. – Data Analytics Internship**
Task 1: Data Immersion & Wrangling  |  Task 2: Exploratory Data Analysis (EDA) & Business Intelligence

---

## About the Project

In this project I took a sales dataset (`Apex1.csv`) with **1,000 orders**, cleaned it in **MySQL Workbench**, and then wrote SQL queries to answer simple business questions like:

- What is the total sales?
- Which products, cities and customers sell the most?
- Which year and month give the best sales?

## Tools Used

- MySQL Workbench (SQL)
- Microsoft Excel (to fix missing ages before import)
- GitHub

## Files in this Repository

| File | What it has |
|------|-------------|
| `Apex1.csv` | The raw dataset |
| `Data_Dictionary.md` | Meaning of every column, plus data quality notes |
| `README.md` | This file – steps and queries |

## Dataset Overview

| Column | Meaning |
|--------|---------|
| Order_ID, Order_Date | Order number and date |
| Customer_ID, Customer_Name, Age, Gender | Customer details |
| City | City of the order |
| Product, Category | What was sold |
| Quantity, Unit_Price, Total_Sales | How many, price of one, and total money |

See `Data_Dictionary.md` for the full details.

---

## Step 1: Importing the Data

- When I first imported the CSV into MySQL, only **980 rows** came in instead of 1,000.
- The reason was missing values in the `Age` column.
- I opened the file in Excel and filled the empty ages with the **average age**.
- After that, all **1,000 rows** were imported successfully.

I then checked the table:

```sql
use customer;
show tables;
desc apex1;
select count(*) from apex1;
select * from apex1;
```

---

## Step 2: Data Cleaning (Task 1)

### 1. Check duplicate records
Group by all columns and find rows that appear more than once.

```sql
select order_id, order_date, customer_id, customer_name, age, gender, city,
       product, category, quantity, unit_price, total_sales, count(*) 'Count'
from apex1
group by order_id, order_date, customer_id, customer_name, age, gender, city,
         product, category, quantity, unit_price, total_sales
having Count > 1;
```

### 2. Check duplicates in single columns and pairs of columns

```sql
-- Order_id
select order_id, count(*) 'Count' from apex1 group by order_id having count > 1;

-- Order_id with customer_name
select order_id, Customer_Name, count(*) 'Count' from apex1
group by order_id, Customer_Name having count > 1;

-- Order_id with customer_id
select order_id, Customer_ID, count(*) 'Count' from apex1
group by order_id, Customer_id having count > 1;

-- Customer_id with customer_name
select customer_name, Customer_ID, count(*) 'Count' from apex1
group by customer_name, Customer_id having count > 1;
```

### 3. Check cities

```sql
select city, count(*) 'num' from apex1 group by city having num > 1;
```

### 4. Replace empty city with "Unknown"

```sql
update apex1 set city = 'Unknown'
where city is null or city = '';
```

### 5. Change the date format and data type

The date was stored as text (`dd-mm-yyyy`). I changed it into a real `DATE`.

```sql
update apex1 set order_date = str_to_date(order_date, '%d-%m-%Y');

alter table apex1 modify column Order_Date date;
```

### 6. Check null values in every column

```sql
select count(*) from apex1 where order_id is null or order_id = '';
select count(*) from apex1 where order_date is null or order_date = '';
select count(*) from apex1 where customer_id is null or customer_id = '';
select count(*) from apex1 where Customer_Name is null or Customer_Name = '';
select count(*) from apex1 where age is null;
select count(*) from apex1 where gender is null or gender = '';
select count(*) from apex1 where quantity is null;
select count(*) from apex1 where unit_price is null;
select count(*) from apex1 where total_sales is null;
```

### 7. Add a primary key column

The table had no primary key, and `order_id` is not unique (same order_id repeats, but the other columns are different). So I added a new `id` column.

```sql
alter table apex1 add column id int primary key auto_increment;
select * from apex1;
```

### 8. Check outliers in Age

First, the minimum, maximum and average age:

```sql
select count(*), min(age) 'Minimum_age', max(age) 'maximum', avg(age) 'average'
from apex1 where age is not null;
```

Then the IQR method (Q1, Q3, lower fence and upper fence):

```sql
with ranked as (
  select age, row_number() over (order by age) as rn,
         count(*) over () as total_rows
  from apex1 where age is not null
),
q as (
  select (select age as q1 from ranked where rn = ceil(0.25 * total_rows)) as q1,
         (select age as q1 from ranked where rn = ceil(0.75 * total_rows)) as q3
)
select q1, q3, q3 - q1 as iqr,
       q1 - 1.5 * (q3 - q1) as lower_fence,
       q3 + 1.5 * (q3 - q1) as upper_fence
from q;
```

---

## Step 3: Analysis – Business Questions (Task 2)

### 1. Total sales

```sql
select sum(total_sales) 'Total' from apex1;
```

### 2. Top 5 selling products

```sql
select product, round(sum(total_sales)) 'Total', count(*) 'Unit'
from apex1 group by product order by total desc limit 5;
```

### 3. Top 10 customers by purchase

```sql
select customer_name, round(sum(total_sales)) 'Total'
from apex1 group by customer_name order by total desc limit 10;
```

### 4. Top 5 selling cities

```sql
select city, round(sum(total_sales)) 'Total'
from apex1 group by city order by total desc limit 5;
```

### 5. Gender-wise total sales

```sql
select gender, round(sum(total_sales)) 'Total' from apex1 group by gender;
```

### 6. Year-wise sales

```sql
select year(order_date) 'Years', round(sum(Total_Sales)) 'Total'
from apex1 group by years order by total desc;
```

### 7. Month-wise sales

```sql
select month(order_date) 'Months', round(sum(total_sales)) 'Total'
from apex1 group by months order by total desc;
```

### 8. Month-wise product selling

```sql
select product, Months, Total
from (
  select product, month(order_date) as Months,
         round(sum(total_sales)) 'Total',
         rank() over (partition by product order by round(sum(total_sales))) as rnk
  from apex1
  group by product, months
) t
where rnk = 1;
```

### 9. Top selling category

```sql
select category, round(sum(total_sales)) 'total'
from apex1 group by category order by total desc;
```

---

## Key Learnings

- How to check and clean data in SQL: duplicates, nulls, wrong data types, outliers
- How to use `GROUP BY`, `HAVING`, `ORDER BY` and `LIMIT`
- How to use date functions (`STR_TO_DATE`, `YEAR`, `MONTH`)
- How to use window functions (`ROW_NUMBER`, `RANK`, `COUNT OVER`) and CTEs
- Missing values can silently drop rows during import – always compare the row count

## Next Steps

- Build a dashboard (Power BI / Excel / Google Sheets) using these results
- Add charts and a KPI mock-up for Task 2
- Write the final summary of insights

---

*Internship project – ApexPlanet Software Pvt. Ltd.*
