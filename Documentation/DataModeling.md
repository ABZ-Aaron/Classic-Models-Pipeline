# Data Modeling

For this project, we will design a dimensional model. 

Our source database in postgres is an operational database. It's fully normalised, and thus optimised for `transactions`. However, our target Data Warehouse needs to be optimised for `analytics`. This is where `Dimensional Modeling` comes in. This technique helps to present data that is `simple to understand for users` and delivers `fast query performance`.

In particular, we will model our data as a `star` or `snowflake` schema, as opposed to our 3rd Normal Form source database. Star schemas include two main components:

1. Fact Tables - this stores performance measurements. For example, a single row in a fact table may represent a single item of an online shopping order, with `facts` or measurements like order line quantity. Other columns in the fact table would likely be foreign keys to lookup or `dimension` tables.
1. Dimension Tables - these tables store context associated with the fact table. For example, you could have a `Date` dimension table, along with a `Product` dimension table.

The star schema is so named because a fact table, surrounded by dimension tables (with lines joining them) resembles a star. A snowflake schema is much the same as a star schema, except that the dimension tables as further normalised. 

## Dimenisional Modeling Steps

The general steps one could follow when creating a dimensional model are:

1. Gather business requirements
1. Choose the business process (i.e., activities performed by organisation)
1. Declare the grain (i.e,. what each record in the fact table will represent)
1. Identify dimensions (i.e., context surrounding the business process event)
1. Identify fact(s) (i.e., measurements tht result from a business process event)

## Designing our Model - Overview

Let's design our dimensional model. 

1. Gathering Business Requirements choosing the Business Process

Because this is a sample project involving the ordering of classical (scale) car models, let's just assume we've had various meetings and determined the business proecss already.

1. Declare the Grain

In our case, a natural grain would be line items on an order, similar to each line on a reciept. Here we are selecting data at it's lowest atomic grain. We could summarise this data by aggregating the line items for a single order - however, we would lose some of the detail here, and users would be unable to drill down if required. That's not to say there aren't case where you would want to create a fact table with summarised data.

1. Identify Dimensions

Now that we know our grain, we can identify relevant dimensions from the data we have available to use that give context to the grain. 

- Order Date
- Shipped Date
- Required Date
- Product
- Customer
- Sales Rep

1. Identify fact(s)

In this case:

- Order Line Quantity 
- Price

We could also `derive` a fact by multiplying order line quantity and price, saving analysts the task of doing this (which could result in greater liklihood of error if they make the wrong calculation).

## Designing our Model - Details

### Date Table

Notice we have 3 date dimensions (order date, shipped date, and required date). We don't have to model 3 seperate date tables, we can just use 1. 

There's a number of columns we could have in our Date table, but we'll keep it simple and only include a handful. Essentially, row in our date table will represent a single day. The columns we'll include are:

- Date Key (primary key)
- Date 
- Day of the Week
- Calendar Month
- Calendar Year
- Weekday Indicator

Unlike other dimension tables, we can make this ahead of time in something like Excel, choosing a time range for it (10 year range, 20 year range, etc).

We can then create a Date table in our target database. Something like:

```SQL
CREATE TABLE Date_Dim (
    DateKey INT,
    FullDate DATE,
    DayOfWeek VARCHAR,
    CalendarMonth VARCHAR,
    CalendarYear INT,
    WeekDayIndicator VARCHAR
);
```

Then populate it from a CSV. Something like:

```SQL
COPY Date_Dim(DateKey, FullDate, DayOfWeek, CalendarMonth, CalendarYear, WeekDayIndicator)
FROM 'C:\date.csv'
DELIMITER ','
CSV HEADER;
```

## A note on Slowly Changing Dimensions

Dimension tables are often static and unchanging, this won't always be the case. Values within a dimension table may change slowly over time. 

For example, a source database may have an `Employee` table with column `address`. If they employee where to relocate, their address would inevitable be updated within this table. When we extract & ingest this data into our database, we need to consider how we handle it. Turns out their are many types of responses to this. Let's look at the 2 most common responses:

1. Type 1 response involves merely overwriting the address within our target database or data warehouse. 

    Although extremely simple to implement, there are some problems here; mainly, we lose all history. If an analyst is running an analysis that requires filtering by employee address location over a fixed period of time and aggregating on some metric, they may end up getting innaccurate results (e.g., assuming an employee has lived in the USA for 4 years, when in fact, they may have only just moved there). This is where Type 2 responses can be useful. 

1. Type 2 response involves maintaing old records when a dimension update occurs, and adding a new record to reflect the change. In practice, you might set this up like so:

    - Add two additional columns to the DW table - `effective date` and `expiration date`. For all current records, effective date would be set to current date, and expiration date would be set to some far off date in the future such as `9999-01-01`. 
    - When a dimension change occurs, such as an address change:
        - the record in question would be updated where `expiration date` would be set to the previous days date
        - a new record would be inserted with the changes, with `effective date` set to current date and `expiration date` set to `9999-01-01`
        - the new record would be given a different new surrogate key

    Through this technique, a history would always be maintained. Analysts would simply need to utilise the `effective date` and `expiration date` when running analytics to ensure they are using the correct record depending on the analysis they are running. 

    Note, you can also add a third `flag` boolean column which indicates which records are `active` (i.e., where current date is between effective and expiration date). This can make filtering easier for downstream users. Also note you can use a combination of Type 1 and Type 2 within the same dimension table.

There are other response types (7 in total), but I won't cover them here. 

### Fact Table

As mentioned, our fact table grain will be order item (single item on an order). The schema of this will be:

- OrderDate (foreign key)
- ShippedDate (foreign key)
- RequiredDate (foreign key)
- ProductCode (foreign key)
- CustomerNumber (foreign key)
- SalesRepKey (foreign key)
- OrderNumber (degenrate dimension)
- OrderLineNumber (degenrate dimension)
- OrderLineQuantity (fact)
- Price (fact)

We've already covered the date dimension, which covers the first 3 columns. Lets look at the remaining dimensions `product`, `customer`, and `sales rep`. 

The `order number` and `order line number` are denerate dimensions and won't have an associated dimension table.

### Product Dimension

You'll notice in our database we have a `productLines` and `products` table. We could utilise a snowflake schema. Here, our dimension table would look more or less the same as the `products` table, with `productLines` being a seperate table joined on to it. 

This seems like it would make sense. The alternative is to flatten de-normalise it, resulting a lot of redudant data. 

However, there are some issues with snowflaking. A couple are:

- Added complexity to overall schema
- Possibly reduced performance (more tables and joins)

The actual disk space savings of snowlflaking are usually insignificant. Taking this into consideration, we will avoid snowflaking, and instead de-normalise the `productLines` and `products` tables, creating one single dimension table:


- ProductKey (PK)
- ProductLine
- ProductLineDescription
- ProductLineHTMLDescription
- ProductLineIMage
- ProductCode
- ProductName
- ProductScale
- ProductVendor
- ProductDescription
- QuantityInStock
- ProductPrice
- MSRP

### Customer Dimension

For customer, one thing to consider is whether sales rep and customer should form the same dimension, or be seperate dimensions. In our case we'll just opt for seperate dimensions as it'll likely be more manageable, and considering that we have very few dimension tables to model anyway. 

However, we will include an attribute or two within the customers table as Type 1 SCDs so that the current sales rep assigned to a customer can be easily identified:

- CustomerKey (PK)
- CustomerName
- ContactFirstName
- ContactLastName
- Phone
- AddressLine1
- AddressLine2
- City
- State
- PostalCode
- Country
- SalesRepEmail
- SalesRepFullName
- SalesRepID
- CreditLimit

### Sales Rep Dimension

For the sales rep dimension, we'll also include details from the `office` table in the source database:

- SalesRepKey (PK)
- EmployeeID
- FirstName
- LastName
- Extension
- Email
- ReportsTo
- JobTitle
- OfficeID
- OfficeCity
- OfficePhone
- OfficeAddressLine1
- OfficeAddressLine2
- OfficeState
- OfficeCountry
- OfficePostalCode
- OfficeTerritory


## Notes

The granularity for our table will be order item. That's to say, each row of our fact table will represent a single item on an order. 

As such, our fact table will have the following columns:

- OrderDate (FK)
- ShippedDate (FK)
- RequiredDate (FK)
- ProductCode (FK)
- CustomerNumber (FK)
- SalesRepKey (FK)
- OrderNumber (DD)
- OrderLineNumber (DD)
- OrderLineQuantity

I've created a Date Dimensional table CSV file. The date key column is just the date in format 'YYYYMMDD'. This isn't to be used by analysts (that could cause performance issues). instead, it can be useufl for partitioning the fact table. For this table I've added dates from 2003 (earliest year in our dataset) up until the end of the current year (2024-2025).

We need to make sure the fact table has no null keys. We can substitue with a word like 'unkonwn'

Consider degenerate dimensions (basically column in fact table that has no dimension table. This could be order number for example)

Unique primary key in a dim table should be a surrogate key (rather than natural key). This is for a handful of reasons:

- Natural keys can be reassigned upstream, which can cause issues to the DE team if they rely on these natural keys.
- Surrogate keys allow DE team to integrate multiple source systems even when they lack consistent natural keys. They can use a mapping file to map multiple natural keys to one common surrogate key
- Improve performance. Because the surrogate key will alawys just be a small integer, it reduces the size and increases the performance of the fact table. 
- Vital for dealing with dimension changes (will cover this later)

Might consider paritioning facgt table on the YYYYMMDD key dimension, and remove old data gracefully. 

We don't really need surrogate key for fact table, but can be useful for immediate identifcation, among other things. 
