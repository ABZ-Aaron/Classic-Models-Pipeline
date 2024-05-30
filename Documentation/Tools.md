# Tools & Services

A range of tools and services were used for this project. Below is a brief description and justification for why each one was chosen. I've also include potential disadvantages of each tool.

```note
NOTE: Most of this does not apply specifically to our project, and is in fact overkill, especially since we're working with small amounts of sample data. The justifications given are working on the assumption that our pipeline is a true production pipeline.
```

## Azure Cloud

The pipeline is primarily built in the cloud using Azure Data Factory & Databricks. There are numerous benefits to this as opposed to running this all on a local machine.

(Potential) Advantages include:

- `Cost Savings` - Assuming one uses the cloud correctly, and takes advantage of all it's features, there are numerous ways in which it can help to save costs. For example, on-premise server infrastructure no longer needs to be maintainined, and servers can be rented when required as opposed to being bought outright and left to run even in times when they are not needed.

- `Scalability` - Computing resources can easily be scaled to meet demands. Require additional compute? A new server can easily be spun-up. 

- `Elasticity` - We can easily configure our cloud resources to scale up or down automatically. For example, if a threshold is met, the system can add or remove servers. Without the cloud, we'd likely have to over-provision (this feeds into cost savings as well)

There are other benefits like making it easier to achieve `high availability` (up-time of our services) and `security`.

It's worth noting some potential disadvantages:

- `Cost Increase` - Although I mentioned cost savings as an advantage, the cloud could actually be more expensive than on-prem solutions if you're not careful. However, this would largely be due to people not understanding the cloud and utilising it incorreclty (e.g., not using auto-scaling features). 

- `Outages` - Although this could also be an issue with on-prem solutions, at least with an on-prem solution, your business will have some level of control over this. If there's a global outage on Azure, there's nothing you can do about it.

For our specific project, the main benefit of the cloud is that we can leave our pipeline to run daily without having to setup our own server, or leave our laptops on overnight. 

## Azure Data Factory

This is a cloud based, low-code/no-code ETL and data integration tool. We use this to extract data from the source system into a raw storage area, and orchestrate our entire pipeline.

This is largely overkill for our project. Databricks actually has something called [workflows](https://docs.databricks.com/en/workflows/index.html) which can be used for orchestration. Using this, we could do everything within Databricks if we wish. ADF was largely chosen for this project to gain experience working with the tool.

However, since we're using ADF, let look at some advantages:

- `No/Low Code` - This may be a disadvantage depending on who you ask, but being low code means that orchestrating pipelines can be made accessible to those with no programming experience.

- `Connectors` - With ADF, we can ingest data from a range or data sources such as AWS Redshift, Google BigQuery, and all Azure data services.

Some disadvantages include:

- `No/Low Code` - Although this is listed as a positive, it can also be viewed as a negative. No code can make building pipelines feel clunky. For basic tasks, ADF is great. Trying to build full pipelines with complex transformations using ADFs `data flows` can be a hassle.


## Azure Data Lake Gen2



## Azure Database for PostgreSQL - Flexible Server

This is a fully managed, cloud based database service. We use this to store our source data, which we'll be treating like an operational, transactional, normalised database.

The primary reason we chose this is that we needed a cloud based database to load our source data into, which we could then extract from using ADF.

Postgres was chosen as the DBMS for no other reason as it's what I'm familar with, and allows me to access the database with tools like PGAdmin. Any DBMS would have been fine for this project.

In a real production environment, there will be vaious factors to consider when choosing a DBMS, e.g., cost.

Regardless, some advantages are:

- `High Availability & Durability` - The database engine runs within a container within a Linux virtual server, with data files seperated and stored on Azure storage (a separation of compute and storage). If enabled, the service maintains a standby server in a separate availability zone within the same region, with data changes on the source server being synchronously replicaed to the standby server. If there is a failure, the standby server will come online and take over. For the data files, the storage essentially maintains three copies  of the files. This all helps to ensure our database remains available, and that we don't lost data.

- `Scalability` - The server comes with a few different compute tiers. We can start off small, and easily adjust the scale to meet our needs.

- `Built-in PgBouncer`- This is sort of like a middle-man that is responsible for managing connections to our database. We can easily enable this if required.

## Databricks

## Delta Tables

## Github

## Power BI

## Flask

