# Azure Services Setup

## Azure Account

For this project, we're using Azure Cloud. You can setup a free account [here](https://azure.microsoft.com/en-gb/free). Please be mindful of what is included in the free account, as it will likely have changed since I setup this project. I'd recommend reading the [Azure Documentation](https://learn.microsoft.com/en-us/azure/azure-portal/azure-portal-overview) or finding something similar if you've never used Azure before.

## [Azure Data Factory](https://azure.microsoft.com/en-gb/products/data-factory)

This is a cloud based, low-code/no-code ETL and data integration tool. We use this to extract data from the source system into a raw storage area, and orchestrate our entire pipeline.

1. In Azure Portal, click on `Create a Resource` and search for `Data Factory`.
1. Select your subscription, and for resource group, click `Create New`, giving it an appropriate name (e.g., `classics-rg`). It's useful to include `rg` at the end so if we see it referenced elsewhere, we can instantly identify what it is. Note that a resource group is basically a container where we can store all our related resources.
1. Select a region close to you. Be mindful that some regions are limited in services.
1. Give the Data Factory an appropriate name (e.g., `classics-adf-dev`). It may be wise to add `dev` (development) to the end. If we setup a CI/CD pipeline, we may create another ADF with `prod` (production) at then end.
1. Everything else can be left as default. 

## [Azure Database for PostgreSQL - Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview) 

This is a fully managed, cloud based database service. We use this to store our source data, which we'll be treating like an operational, transactional, normalised database.

To create an instance of this:

1. Search for `Azure Database for Postgres flexible servers`
1.  Click `Create`
1. Most options are are self-explanatory and can be left as default. One you will want to change is for `Workload Type` select `Development`. This will change the computer + storage assigned. It is designed for personal projects and will be the cheapest option.
1. Give an appropriate `admin name` + `password` and note these down.
1. For Firewall Rules, allow public access from any Azure Services, add your public IP address (found [here](https://whatismyipaddress.com)) to list of firewall rules. You may need to change this periodically if your public IP changes.
1. Before creating the server, be sure to look at the `Estimated Costs` box on the right hand side. Your estimated total should be somewhere around $20 / Month (at time of writing this) if you've configured it with the cheaper options. 
1. Under `Server Parameters`, turn off `required_secure_transport`.
1. Go to Databases in Azure Postgres, click Add. Call it `classicmodels`
1. Install [psql](https://www.timescale.com/blog/how-to-install-psql-on-mac-ubuntu-debian-windows/) locally. Run following command (replacing server name) to create tables in Database. Note, server name will be something like `azure-postgres-flexible.postgres.database.azure.com` and can be found on the home page of your Postgres instance. You can read comment at the top of the SQL file to better understand what it does.

```bash
psql -h [server name] -p 5432 -U postgres -d classicmodels -f ./SQL/create-classics-database.sql
```

## [Azure Data Lake Gen2 Setup](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)

This is essnetially blob stroage with some big data analytics capabilities (e.g., file system semantics to make organising data easier). Blob storage is Azure's object storage solution, which is optimised for storing large amounts of data while remaining cost-efficient. We use this to store all of our data after the initial extraction.


1. Search for `Storage Accounts` and click `Create`
1. Most options are self-explanatory and left as default.
1. Give it a unique name (e.g., `classicsdatalake`)
1. Select `locally-redundant storage LRS`. This is lowest cost option.
1. On Advanced Tab, select `Enable Heirarchical Namespace`

>NOTE: If you would like to access this storage account from your local machine without having to go through the Azure Portal, you can install Azure Storage Browser from [here](https://learn.microsoft.com/en-us/azure/vs-azure-tools-storage-manage-with-storage-explorer?tabs=macos).

## [Azure KeyVault](https://azure.microsoft.com/en-gb/products/key-vault)

This is essentially a password and secrets manager in the cloud. We use this to store passwords for some of our cloud services (e.g., our Postgres password) which can then be accessed by other cloud services securely. 

1. Search for `Key Vaults`  and click `Create`
1. Most options can be left as default.
1. Give it a name, e.g., `classics-keyvault`
1. Give it an appropriate Region.
1. To `Access control (IAM)` and click on `Add Role Assignment`
1. Here you want to select Key Vault Administrator, then on Members, select Managed Identity, and search for your ADF.
1. You also want to add yourself as a member.
1. Create a new secret called PostgreSQLPassword, and put the password for your postgres instance in.

## Configurations

### Azure Data Factory

1. Go to `Linked Services` and search for `Key Vault`
1. Here you want to select the KeyVault you created previously. Put `_ls` at the end of the name so we can easily recognise it as a linked service.
1. Once added, click `Publish All` at the top to save these changes.
1. Add another linked service, for Azure Data Lake Storage Gen2. You'll want to follow the same steps for your storage account as you did for the KeyVault (create role etc)
1. For this linked service, change from access key to system managed identify. With managed identities, these are f