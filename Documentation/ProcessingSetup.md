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

## Databricks

- Create new Databricks instance
- Add to relevant resource group
- For pricing teir, select Standard
- Leave rest as default
- Go to resource once created
- Click Create Compute from left hand menu. Here we will create an all-purpose Databricks Cluster.
- Click create cluster
    - Change to single node cluster
    - Leave performance as default
    - Set to terminate cluster after 10 or 20 minutes. This will help save costs.
    - For Node type, leave this as default. Make sure it's the lowest general purpose node type to save costs.
- Leave rest as default and create cluster

- Create service principlle
    - Search Entra ID (formerly active directory)
    - Select App Registration and click create new registration
    - Call it classics-app and register
    - Take a note of application ID and directory ID
    - Finally, go to Secrets on the left and create a new Client Secret. Once created, take a note of the Value.
    - Go to our data lake stroage account, click IAM, and click Add new role assignment.
    - Select storage blob contributer role
    - Under members, search and add classics app
    - Review and assign. 
    - What we've done here is allowed our service principle contribuer access to our storage account
    s- In Databricks, under workspace, create a folder `Classics`, and in there create another folder `Setup`