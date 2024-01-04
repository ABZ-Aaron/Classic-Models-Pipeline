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

I've created a Date Dimensional table CSV file. The date key column is just the date in format 'YYYYMMDD'. This isn't to be used by analysts (that could cause performance issues). instead, it can be useufl for partitioning the fact table

We need to make sure the fact table has no null keys. We can substitue with a word like 'unkonwn'

Consider degenerate dimensions (basically column in fact table that has no dimension table. This could be order number for example)

Unique primary key in a dim table should be a surrogate key (rather than natural key). This is for a handful of reasons:

- Natural keys can be reassigned upstream, which can cause issues to the DE team if they rely on these natural keys.
- Surrogate keys allow DE team to integrate multiple source systems even when they lack consistent natural keys. They can use a mapping file to map multiple natural keys to one common surrogate key
- Improve performance. Because the surrogate key will alawys just be a small integer, it reduces the size and increases the performance of the fact table. 
- Vital for dealing with dimension changes (will cover this later)

Might consider paritioning facgt table on the YYYYMMDD key dimension, and remove old data gracefully. 

We don't really need surrogate key for fact table, but can be useful for immediate identifcation, among other things. 
