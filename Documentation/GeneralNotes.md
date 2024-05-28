# To Do List

handle late-arriving data
keep raw data to support reprocessing
handle occasional bursts from reprocessing, backfilling, etc
support comprehensive unit-testing and constraint-checking
minimize tight-coupling by subscribing to domain objects rather than replicate physical schemas
manage costs on cloud databases
provide a low-enough data latency to satisfy their users
automate everything
observability & alerting & KPIs
data dictionaries / data catalogs / documentation (https://www.castordoc.com/blog/what-is-a-data-dictionary)
write ahead log?

# General Notes


For KeyVault, make sure permission model is set to Vault Access Policy. If use Azure role badsed access control, give Key vault admin access to service principle.


DBFS = Databricks file system

Dbutils 

- ls
- ls /databricks-datasets
- dbutils.fs.ls('/') 

Azure storage account comes with storage access keys we can use to access it. We can also generate SAS tokens. We can also create a service principle. We can give required access to data lake to service principle and use those credentials to access storage account. 

Service Principle 

- Similar to user accounts. Registered in Azure Active Directory. Assigned permissions to access Azure resources using role based access control.
- Recommend for automated tools
- First create the SP (azure ad application)
- Given a unique identifier
- Also need to create secret for SP
- Configure Databricks to access storage account using SP
- Then assign required role on the data lake for the service principle so service principle can access the data lake.
- We give contributor role access for our storage account to the service principle.

- use secret scope in databricks. This helps store credentials security and reference them in notebooks. We will use azure key-vault backed secret scope. Here we keep secrets in key-vault and share them with databricks. 

So we crete azure-key vault and link databricks to secret scope. Then use the databricks utility to access them.

Create secrets for client, tendent, and client secrets.

We then create databricks secret scope
- In databricks, go to homepage. Then change append the following to the url to access the 'hidden' area where you will create a secret scope

    #secrets/createScope

- Give a scope name (classics-scope)
- In KeyValut, click on properties and take note of:
    - Vault ID
    - Resource ID
- Back in Secret Scropes enter these into DNS Name and Resource ID respectively 
- See notebook Access Azure Data lake using Service Principle to see how to use databricks secret utility to access secret

- HOWEVER - won't use this method exactly. We'll use mounts:
 - Databricks File System (DBFS) is a distirubted file system mounted onto databricks. It provides distributed access to data stored in azure storaage. Default location for managed tables is dbfs root. Not recommended for our data however. Instead, we use our external datalake and mount that to dbfs. 

 - We can then access mount points without using credentials. 

 - Add indexes to Data Warehouse once complete. 

 # Databases

 In order for Spark to treat our data as tables and columns, we need to register this data 
 in a metastore. Spark uses Hive Meta Store to store this metadata. 

 - Managed Tables (spark manages the metadata and data files)
 - External Tables (spark manages the metadata and we manage the data files)


Cluser in databricks should be created as No Isolation Shared, which will allow ADF managed identity to access the cluster.

