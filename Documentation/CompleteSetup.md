# Complete Setup

## Services

### Azure Account

For this project, we're using Azure Cloud. You can setup a free account [here](https://azure.microsoft.com/en-gb/free).

Please be mindful of what is included in the free account, as it will likely have changed since I setup this project.

I'd recommend reading the [Azure Documentation](https://learn.microsoft.com/en-us/azure/azure-portal/azure-portal-overview) or finding something similar if you've never used Azure before.

### Azure Data Factory

1. In Azure Portal, click on `Create a Resource` and search for `Data Factory`.
1. Select your subscription, and for resource group, click `Create New`, giving it an appropriate name (e.g., `classics-rg`). It's useful to include `rg` at the end so if we see it referenced elsewhere, we can instantly identify what it is. Note that a resource group is basically a container where we can store all our related resources.
1. Select a region close to you.
1. Give the Data Factory an appropriate name (e.g., `classics-adf-dev`). It may be wise to add `dev` (development) to the end. If we setup a CI/CD pipeline, we may create another ADF with `prod` (production) at then end.
1. Everything else can be left as default. 
1. Once the ADF is created, go to resource and launch ADF.

### Azure Database for PostgreSQL - Flexible Server

Postgres [Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview) is a fully managed database service provided by Azure, and is considered a more improved version over Single Server. See [here](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-compare-single-server-flexible-server#comparison-table) for comparisons.

Some advantages it provides over Single Server are cost optimisation controls, predictable performance, and a higher level of control. 

To create an instance of this:

1. Search for `Azure Database for Postgres flexible servers`
1. Click `Create`
1. Most options are are self-explanatory and can be left as default. One you will want to change is for `Workload Type` select `Development`. This will change the computer + storage assigned. It is designed for personal projects and will be the cheapest option.
1. Give an appropriate `admin name` + `password` and note these down.
1. For Firewall Rules, allow public access from any Azure Services, add your public IP address (found [here](https://whatismyipaddress.com)) to list of firewall rules. You may need to change this periodically if your public IP changes.
1. Before creating the server, be sure to look at the `Estimated Costs` box on the right hand side. You're estimated total should be somewhere around $20 / Month (at time of writing this) if you've configured it with the cheaper options. 
1. Under `Server Parameters`, turn off `required_secure_transport`.
1. Go to Databases in Azure Postgres, click Add. Call it `classicmodels`
1. Install [psql](https://www.timescale.com/blog/how-to-install-psql-on-mac-ubuntu-debian-windows/) locally. Run following command (replacing server name) to create tables in Database:

```bash
psql -h [server name] -p 5432 -U postgres -d classicmodels -f ./SQL/create-classics-database.sql
```

Note, server name will be something like `azure-postgres-flexible.postgres.database.azure.com` and can be found on the home page of your Postgres instance.

The above command connects to your database and runs the `create-classics-database` SQL script. This script:

1. creates the database mentioned [here](https://www.mysqltutorial.org/getting-started-with-mysql/mysql-sample-database/) with some slight modifications to allow it to work with Postgres and to give each table a `last_updated` column.
1. creates function which updates the `last_updated` column for each table with current timestamp when called.
1. creates triggers on each table, so that when a change is made to a record on a table, the function to update the `last_updated` column with current timestamp will be called. This is important, as for each pipeline run, it allows us to determine which records in the database were last updated since the last pipeline run - allowing us to incrementally extract data.
1. creates configurations `control` table and populates it. This table contains metadata regarding our database tabes, such as table names, schema name, column used to determine last update of table records, etc. Importantly, it contains a `FromDate` column. Using this column, we take a `WaterMark` approach to incremental extraction. Essentially, after each pipeline run, we will set this value to the maximum `last update` timestamp. Then on the next run, our process knows to only extract records greater than this timestamp. You can read more about this [here](https://datakuity.com/2022/12/14/incrementally-load-with-synapse/). We follow a very similar approach.
1. Creates procedure which will update watermark value in control table with passed timestamp.


>NOTE: If you want to access this database from your local machine with a GUI, you can install Azure Data Studio from [here](https://learn.microsoft.com/en-us/azure-data-studio/download-azure-data-studio?tabs=win-install%2Cwin-user-install%2Credhat-install%2Cwindows-uninstall%2Credhat-uninstall)

### Azure Data Lake Gen2 Setup

This is built on top of of the standard Azure storage account, but provides some additional capabilities such as heirarchical namespace and big data compatibility. You can read more [here](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction).

1. Search for `Storage Accounts` and click `Create`
1. Most options are self-explanatory and left as default.
1. Give it a unique name (e.g., `classicsdatalake`)
1. Select `locally-redundant storage LRS`. This is lowest cost option.
1. On Advanced Tab, select `Enable Heirarchical Namespace`

>NOTE: If you would like to access this storage account from your local machine without having to go through the Azure Portal, you can install Azure Storage Browser from [here](https://learn.microsoft.com/en-us/azure/vs-azure-tools-storage-manage-with-storage-explorer?tabs=macos).

### Azure KeyVault

1. Search for `Key Vaults`  and click `Create`
1. Most options can be left as default.
1. Give it a name, e.g., `classics-keyvault`
1. Give it an appropriate Region.
1. To `Access control (IAM)` and click on `Add Role Assignment`
1. He you want to select Key Vault Administrator, then on Members, select Managed Identity, and search for your ADF.
1. You also want to add yourself as a member.
1. Create a new secret called PostgreSQLPassword, and put the password for your postgres instance in.

## Configurations

### Azure Data Factory

1. Go to `Linked Services` and search for `Key Vault`
1. Here you want to select the KeyVault you created previously. Put `_ls` at the end of the name so we can easily recognise it as a linked service.
1. Once added, click `Publish All` at the top to save these changes.
1. Add another linked service, for Azure Data Lake Storage Gen2. You'll want to follow the same steps for your storage account as you did for the KeyVault (create role etc)
1. For this linked service, change from access key to system managed identify. With managed identities, these are 