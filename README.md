## Step 1: Environment Setup
  This initial setup ensures secure credential management while setting up the foundational database environment for the pipeline
### Download and Restore Database

- Downloaded AdventureWorksLT2017.bak, an OLTP database sample, from the official Microsoft website.
- Restored the .bak file in SQL Server Management Studio (SSMS) and opened the database for further configuration.

### User Login Creation

- Created a new user login within the SQL Server database using SQL queries.This user was assigned specific credentials (username and password) and granted a db_datareader role for database access.

  `CREATE LOGIN 'username' WITH PASSWORD = 'password'
    create user 'username' for login 'username'`
  
  <img width="400" alt="image" src="https://github.com/user-attachments/assets/853cf2f8-e112-45a8-ac1f-f434b845fe05">



### Setting Up Azure Key Vault for Secure Credential Management

- Created an Azure Key Vault in the Azure portal to securely store database credentials for later use.
- Added two secrets in Key Vault:
  - Username Secret: Stored the username as a secret.
  - Password Secret: Stored the password as a separate secret for added security.

## Step 2: Data Ingestion
### Create Azure Data Factory (ADF)
- Created an Azure Data Factory (ADF) resource in the Azure portal to manage and automate data workflows.
  
- Setup for On-Premises SQL Server Access
  - Installed a Self-Hosted Integration Runtime (SHIR) to enable ADF access to the on-premises SQL Server.
 
- Create Data Lake Storage Gen2 for Data Storage
  - Created a Data Lake Storage Gen2 resource to store ingested data.
    
- Configure ADF Pipeline for Data Ingestion
  - Configured a Lookup activity to query all tables in the SalesLT schema from the AdventureWorksLT2017 database.
    `SELECT s.name as schemaName
    t.name as tableName
    from sys,table t
    join sys,schema as s on t.schema_id = s.schema_id
    when s.name = 'SaleLT'`
    
  - Created a linked service for SQL Server, connected to the on-premises server through SHIR, and authenticated using Key Vault secrets.
    
- Iterate Over Tables Using ForEach Activity
  - Used a ForEach activity to iterate over the output of the Lookup activity, retrieving each table one by one.
  - Within ForEach, added a Copy Data activity to copy each table to Data Lake Storage Gen2.
  - For the Copy Data activity:
    - Configured the source data using the SQL Server linked service via SHIR.
    - Set up a sink by creating a new linked service for Data Lake Storage Gen2.
    - Chose .parquet format for table storage with a structured file path: `bronze/salesLT/'tablename'/'tablename'.parquet`

## Step 3: Data Transformation
### Set Up Databricks for Data Transformation

- Created a Spark compute cluster in Azure Databricks with credential passthrough enabled for Data Lake access.
- Granted Blob Data Contributor role in Data Lake Storage to allow access.
  
### Mount Data Lake Storage to Databricks

- Created a storage_mount notebook to mount data from Azure Data Lake Storage Gen2 to Databricks.
- Configured a Bronze container for raw data, and created Silver and Gold containers to follow the medallion architecture.

### Bronze to Silver Transformation

- Created a bronze_to_silver notebook to process data from the Bronze container.
- Since the source data was already relatively clean, minimal transformation was needed. The main transformation was converting datetime columns to date format.
- Saved the transformed data in the Silver container in .delta format.

### Silver to Gold Transformation

- Created a silver_to_gold notebook to process data from the Silver container.
- Renamed column names from PascalCase to snake_case to adhere to business naming conventions.
- Stored the transformed data in the Gold container in .delta format.

### Automate Notebooks in ADF Pipeline

- Deleted the storage_mount notebook to avoid remounting data in each pipeline run.
- Added a new linked service in Azure Data Factory to connect to Databricks, using an access token, which was stored as a secret in Key Vault.

### Add Transformation Notebooks to ADF Pipeline

- Configured two Notebook activities in the ADF pipeline: bronze_to_silver and silver_to_gold.
- Connected both notebooks with the ForEach activity to automate transformations as data flows through the pipeline.

   


