# Data Processing

Now that we've extracted data to our Raw container, we can run some transformations to clean and conform it, before copying it to our Processed container.

In our case, we'll more-or-less be creating our fact and dimension tables ready for loading into our final Presentation layer. 

We'll follow a similar approach outlines [here](https://iterationinsights.com/article/how-to-implement-slowly-changing-dimensions-scd-type-2-using-delta-table/) which will allow us to deal with both SCD1 and SCD2.  

Here's the transformations we'll run:

- Create Surrogate Key columns, which will be empty at this stage
- Create Effective Date, End Date, and Flag columns with default values (Flag set to True, Effective Date set to current date, and End Date set to a distant, far off date.)
- Create Hash column, which will be empty at this stage. This will be generated from the columns in the dimension table we've marked as SCD2, to tell us if any of those columns differ from what's in our Presentation (kimball model) layer.
- Join relevant tables (e.g., Employees and Offices to create Sales Rep table)
- De-duplicate data. It may be that the extraction pipeline was run multiple times in a day, and potentially captured multiple changes to a single record for that day. For simplicity, we'll just remove the older record.
- Rename columns
- Remove some irrelevant columns

