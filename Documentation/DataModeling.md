# Data Modeling

For this project I've designed a dimensional model.

The source database in postgres is an operational database. It's fully normalised and thus optimised for `transactions`. However, the target Data Warehouse should to be optimised for `analytics`. This is where `Dimensional Modeling` comes in. This technique helps to present data that is `simple to understand for users` and delivers `fast query performance`.

In particular, I've modelled the data as a `star` schema, as opposed to the 3rd Normal Form source database. Star schemas include two main components:

1. Fact Tables - this stores performance measurements. For example, a single row in a fact table may represent a single item of an online shopping order, with `facts` or measurements like order line quantity. Other columns in the fact table would likely be foreign keys to lookup or `dimension` tables.
1. Dimension Tables - these tables store context associated with the fact table. For example, you could have a `Date` dimension table, along with a `Product` dimension table.

The star schema is so named because a fact table, surrounded by dimension tables (with lines joining them) resembles a star. A snowflake schema is much the same as a star schema, except that the dimension tables as further normalised. 

When designing a dimensional model, you'll want to layout the design somewhere, map source columns to target columns, etc. In our case, a basic excel file has been put together with details on each table [here](Documentation/TableDesign.xlsx).

## Dimenisional Modeling Steps

The general steps one could follow when creating a dimensional model are:

1. Gather business requirements
1. Choose the business process (i.e., activities performed by organisation)
1. Declare the grain (i.e,. what each record in the fact table will represent)
1. Identify dimensions (i.e., context surrounding the business process event)
1. Identify fact(s) (i.e., measurements that result from a business process event)

## Designing the Model - Overview

1. Gathering Business Requirements and choosing the Business Process

Because this is a sample project involving the ordering of classical (scale) car models, we can assume we've had various meetings and determined the business process already.

1. Declare the Grain

In our case, a natural grain would be line items on an order, similar to each line on a reciept. Here we are selecting data at its lowest atomic grain. We could summarise this data by aggregating the line items for a single order - however, we would lose some of the detail here, and users would be unable to drill down if required. That's not to say there aren't case where you would want to create a fact table with summarised data.

As our sample database also has customer payment information, we can further track this within another fact table, with grain being payments made by each customer.

1. Identify Dimensions

Now that we know the grain for both of our fact tables, we can identify relevant dimensions from the data we have available to use that give context to these. 

For our Order Line Items table:

- Order Date
- Shipped Date
- Required Date
- Product
- Customer
- Sales Rep

For our Payments table:

- Customer
- Payment Date

1. Identify fact(s)

For our Order Line Items table:

- Order Line Quantity 
- Price

For our Payments table:

- Payment Amount

We could also `derive` a fact by multiplying order line quantity and price, saving analysts the task of doing this (which could result in greater liklihood of error if they make the wrong calculation). Additionally, we can add `degenerate dimesnions` that have no dimension table like order number and order line item number. 

## Designing the Model - Surrogate Keys

For our dimension tables, we'll include numeric surrogate (artificial) keys. We could rely on the natural keys from the source system (e.g., customer ID). However, there are several disadvantage to this. A couple of these are:

- We'll be using Slowly Changing Dimensions (SCD) type 2 (discussed later). In short, this allows us to track history within our dimension tables by essentially marking existing dimension records as inactive and inserting the new, updated record. In such cases, we could end up with two or more duplicate natural keys within a dimension.
- In some cases, a source system may begin recycling natural keys.
- A natural key can be composed of various letters and symbols. As our fact table will be largely populated with foreign keys using these natural keys, and could have a signficant amount of records, we can save space by replacing these keys with simple integer surrogate keys.

## Designing the Model - Details

### Date Table

Notice in our Order Line Items table we have 3 date dimensions (order date, shipped date, and required date). We don't have to model 3 seperate date tables, we can just use 1. 

There's a number of columns we could have in our Date table, but we'll keep it simple and only include a handful. Essentially, each row in our date table will represent a single day. The columns we'll include are:

- Date Key (primary key)
- Date 
- Day of the Week
- Calendar Month
- Calendar Year
- Weekday Indicator

Unlike other dimension tables, we can make this ahead of time in something like Excel, choosing a time range for it (10 year range, 20 year range, etc).

We can then create a Date table in our target database. Something like:

```SQL
CREATE TABLE date_dim (
    date_key INT,
    full_date DATE,
    day_of_week INT,
    calendar_month VARCHAR,
    calendar_year INT,
    weekday_indicator VARCHAR
);
```

Then populate it from a CSV. Something like:

```SQL
COPY date_dim(date_key, full_date, day_of_week, calendar_month, calendar_year, weekday_indicator)
FROM 'C:\date.csv'
DELIMITER ','
CSV HEADER;
```

I've created a Date Dimensional table CSV file [here](SQL/date_dim.csv). The date key column is just the date in format 'YYYYMMDD'. This isn't to be used by analysts (that could cause performance issues). instead, it can be useufl for partitioning the fact table. For this table I've added dates from 2003 (earliest year in our dataset) up until the end of the current year (2024-2025).

## A note on Slowly Changing Dimensions

Dimension tables are often static and unchanging, this won't always be the case. Values within a dimension table may change slowly over time. 

For example, a source database may have an `Employee` table with column `address`. If they employee where to relocate, their address would inevitable be updated within this table. When we extract & ingest this data into our database, we need to consider how we handle it. Turns out their are many types of responses to this. Let's look at the 2 most common responses:

1. Type 1 response involves merely overwriting the address within our target database or data warehouse. 

    Although simple to implement, there are some problems here, such as losing all history. If an analyst is running an analysis that requires filtering by employee address location over a fixed period of time and aggregating on some metric, they may end up getting innaccurate results (e.g., assuming an employee has lived in the USA for 4 years, when in fact, they may have only just moved there). This is where Type 2 responses can be useful. 

1. Type 2 responses involve maintaing old records when a dimension update occurs, and adding a new record to reflect the change. In practice, you might set this up like so:

    - Add two additional columns to the DW table - `effective date` and `expiration date`. For all current records, effective date would be set to current date, and expiration date would be set to some far off date in the future such as `9999-01-01`. 
    - When a dimension change occurs, such as an address change:
        - the record in question would be updated where `expiration date` would be set to the previous days date
        - a new record would be inserted with the changes, with `effective date` set to current date and `expiration date` set to `9999-01-01`
        - the new record would be given a different surrogate key

    Through this technique, a history would always be maintained. Analysts would simply need to utilise the `effective date` and `expiration date` when running analytics to ensure they are using the correct record depending on the analysis they are running. 

    Note, you can also add a third `flag` boolean column which indicates which records are `active` (i.e., where current date is between effective and expiration date). This can make filtering easier for downstream users. Also note you can use a combination of Type 1 and Type 2 within the same dimension table.

There are other response types (7 in total), but I won't cover them here. For our project, we will not look too in depth here. However, we will implement SCD1 and SCD2 for some attributes to show how it might be done.

### Fact Table

As mentioned, our fact table grain will be order item (single item on an order). The schema of this will roughly be:

- Order Date Key (foreign key)
- Shipped Date Key (foreign key)
- Required Date Key (foreign key)
- Product Key (foreign key)
- Customer Key (foreign key)
- Sales Rep Key (foreign key)
- Order Number (degenrate dimension)
- Order Line Number (degenrate dimension)
- Order Line Quantity (fact)
- Price (fact)

We've already covered the date dimension, which covers the first 3 columns. Lets look at the remaining dimensions `product`, `customer`, and `sales rep`. We did mention a `payments` dimension, however, we'll join this to the `customers` dimension.

The `order number` and `order line number` are denerate dimensions and won't have an associated dimension table. The order number in particular might be useful, as it would allow analysts to group order lines by order number.

### Product Dimension

You'll notice in our database we have a `productLines` and `products` table. We could take a snowflake approach. Here, our dimension table would look more or less the same as the `products` table, with `productLines` being a seperate table joined on to it. 

This seems like it would make sense. The alternative is to de-normalise it, resulting a lot of redudant data. 

However, there are some issues with snowflaking. A couple are:

- Added complexity to overall schema
- Possibly reduced performance (more tables and joins)

The actual disk space savings of snowlflaking are usually insignificant. Taking this into consideration, we will avoid snowflaking, and instead de-normalise the `productLines` and `products` tables, creating one single dimension table:


- Product Key (PK)
- Product Line
- Product Line Description
- Product Code
- Product Name
- Product Scale
- Product Vendor
- Product Description
- Quantity In Stock
- Product Price
- MSRP

We've ignored the HTML description and image columns from the source database.

### Customer Dimension

- Customer Key (PK)
- Customer Number
- Customer Name
- Contact First Name
- Contact Last Name
- Phone
- Address Line 1
- Address Line 2
- City
- State
- Postal Code
- Country
- Sales Rep Email
- Sales Rep Full Name
- Sales Rep ID
- Credit Limit

### Sales Rep Dimension

For the sales rep dimension, we'll also include details from the `office` table in the source database:

- Sales Rep Key (PK)
- Employee ID
- First Name
- Last Name
- Extension
- Email
- Reports To
- Job Title
- Office ID
- Office City
- Office Phone
- Office Address Line 1
- Office Address Line 2
- Office State
- Office Country
- Office Postal Code
- Office Territory

### Order status

```
NOT CURRENTLY TRACKING
```

## Dimensional Model Schema

<img src="https://github.com/ABZ-Aaron/Classic-Models-Pipeline/blob/master/Documentation/Images/DMSchema.png" width=70% height=70%>

If you'd like to create this yourself, use [dbdiagram.io](https://dbdiagram.io) and paste in the following code.

```
Table order_line_item_fct as fact {
  order_date_key int
  shipped_date_key int
  required_date_key int
  product_key int
  customer_key int
  sales_rep_key int
  order_number int
  order_line_number int
  order_line_quantity int
  price numeric
}

Table date_dim {
  date_key int [pk]
  full_date date
  day_of_week int
  calendar_month varchar(3)
  calendar_year int
  weekday_indicator varchar(7)
}

Table product_dim {
  product_key int [pk]
  product_line varchar(50)
  product_line_description varchar(4000)
  product_code varchar
  product_name varchar
  product_scale varchar
  product_vendor varchar
  product_description varchar
  quantity_in_stock int
  product_price numeric
  msrp numeric
}

Table customer_dim {
  customer_key int [pk]
  customer_number int
  payment_key int
  customer_name varchar
  contact_first_name varchar
  contact_last_name varchar
  phone varchar
  address_1 varchar
  address_2 varchar
  city varchar
  state varchar
  postal_code varchar
  country varchar
  sales_rep_email varchar
  sales_rep_fullname varchar
  sales_rep_id int
  credit_limit numeric
}

Table payment_fct {
  customer_key int
  payment_date_key int
  check_number int
  amount numeric
}

Table sales_rep_dim {
  sales_rep_key int [pk]
  sales_rep_id int
  first_name varchar
  last_name varchar
  extension varchar
  email varchar
  reports_to int
  job_title varchar
  office_id int
  office_city varchar
  office_phone varchar
  office_address_1 varchar
  office_address_2 varchar
  office_state varchar
  office_country varchar
  office_postal_code varchar
  office_territory varchar
}

Ref: fact.order_date_key > date_dim.date_key
Ref: fact.shipped_date_key > date_dim.date_key
Ref: fact.required_date_key > date_dim.date_key
Ref: fact.product_key > product_dim.product_key
Ref: fact.customer_key > customer_dim.customer_key
Ref: fact.sales_rep_key > sales_rep_dim.sales_rep_key


Ref: "payment_fct"."customer_key" < "customer_dim"."customer_key"
Ref: "payment_fct"."payment_date_key" < "date_dim"."date_key"
```