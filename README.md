<img width="500" alt="azureawltss" src="https://github.com/user-attachments/assets/672177b9-1a1b-4643-9a8a-ba109642eee7">


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

`configs = { "fs.azure.account.auth.type": "CustomAccessToken", "fs.azure.account.custom.token.provider.class": spark.conf.get("spark.databricks.passthrough.adls.gen2.tokenProviderClassName") }
dbutils.fs.mount( source = "abfss://bronze@<storage-account-name>.dfs.core.windows.net/", mount_point = "/mnt/bronze", extra_configs = configs )`

`configs = { "fs.azure.account.auth.type": "CustomAccessToken", "fs.azure.account.custom.token.provider.class": spark.conf.get("spark.databricks.passthrough.adls.gen2.tokenProviderClassName") }
dbutils.fs.mount( source = "abfss://silver@<storage-account-name>.dfs.core.windows.net/", mount_point = "/mnt/silver", extra_configs = configs )`

`configs = { "fs.azure.account.auth.type": "CustomAccessToken", "fs.azure.account.custom.token.provider.class": spark.conf.get("spark.databricks.passthrough.adls.gen2.tokenProviderClassName") } 
dbutils.fs.mount( source = "abfss://gold@<storage-account-name>.dfs.core.windows.net/", mount_point = "/mnt/gold", extra_configs = configs )`
  
- Configured a Bronze container for raw data, and created Silver and Gold containers to follow the Lake House architecture.

### Bronze to Silver Transformation

- Created a bronze_to_silver notebook to process data from the Bronze container.
- Since the source data was already relatively clean, minimal transformation was needed. The main transformation was converting datetime columns to date format.
- Saved the transformed data in the Silver container in .delta format.

`table_name = [] 
for i in dbutils.fs.ls('/mnt/bronze/SalesLT/'): 
table_name.append(i.name.split('/')[0])
from pyspark.sql.functions import from_utc_timestamp, date_format from pyspark.sql.types import TimestampType
for i in table_name:
path = '/mnt/bronze/SalesLT/' + i + '/' + i + '.parquet'
df = spark.read.format('parquet').load(path) 
column = df.columns
for col in column: 
if "Date" in col or "date" in col:
df = df.withColumn(col, date_format(from_utc_timestamp(df[col].cast(TimestampType()), "UTC"), "yyyy-MM-dd"))
output_path = '/mnt/silver/SalesLT/' + i + '/'
df.write.format('delta').mode("overwrite").save(output_path)`

### Silver to Gold Transformation

- Created a silver_to_gold notebook to process data from the Silver container.
- Renamed column names from PascalCase to snake_case to adhere to business naming conventions.
- Stored the transformed data in the Gold container in .delta format.

`table_names=[]
for i in dbutils.fs.ls("mnt/silver/SalesLt')
table_names.append(i.name.split('/')[0])
for name in table_name: path = '/mnt/silver/SalesLT/' + name print(path) df = spark.read.format('delta').load(path) # Get the list of column names column_names = df.columns for old_col_name in column_names: 
new_col_name = "_".join(["_" + char if char.isupper() and not old_col_name[i - 1].isupper() else char for i, char in enumerate(old_col_name)]).lstrip("_") 
df = df.withColumnRenamed(old_col_name, new_col_name) output_path = '/mnt/gold/SalesLT/' + name + '/' df.write.format('delta').mode('overwrite').save(output_path)`

### Automate Notebooks in ADF Pipeline

- Deleted the storage_mount notebook to avoid remounting data in each pipeline run.
- Added a new linked service in Azure Data Factory to connect to Databricks, using an access token, which was stored as a secret in Key Vault.

### Add Transformation Notebooks to ADF Pipeline

- Configured two Notebook activities in the ADF pipeline: bronze_to_silver and silver_to_gold.
- Connected both notebooks with the ForEach activity to automate transformations as data flows through the pipeline.

## Step 4: Data Loading
### Set Up Azure Synapse Analytics

- Created an Azure Synapse Analytics resource to facilitate querying and integration with the Data Lake.

### Configure Serverless SQL Database

- Set up a serverless SQL database within Synapse using the built-in serverless SQL pool for computation.
- This configuration ensures that any changes in the original Data Lake database automatically reflect in the Synapse SQL database.

### Access Data Lake from Synapse

- Established access to Data Lake containers using a linked service with Azure SQL Server and System Assigned Managed Identity.
- Copied the endpoint of the Synapse SQL database for later connections.
- Created a view for the Gold container and a stored procedure to facilitate loading data into the Synapse SQL database.

`Use gold_DB
Go
Create or Alter Proc CreateSQLserverlessView_Gold
@ViewName nvchar(100)
As
Begin`

`Declare @statement Varchar(max)
    Set @statement=N'Create or alter view' + @ViewName +' 
    As
    Select * from
    Openrowset( 
    bulk "https://goldcontainer + @ViewName + '/'", format="Delta"
    ) as [result]`
    
`Exec(@statement)
End
Go`

### Build Data Loading Pipeline in Azure Synapse

- From the Integrate tab, created a pipeline with the following activities:

### Get Metadata Activity:

- Fetched all table names from the Gold container using a linked service with Azure Data Lake Gen2.
- Retrieved data in .binary format from gold/salesLT.
- Configured the activity to select Child Items in the field list to get all table names.

### ForEach Activity:

- Iterated over the array of child items retrieved by the Get Metadata activity.
- Inside the ForEach Activity, added a Stored Procedure Activity:
- Referenced the stored procedure created in the serverless SQL database.
- Connected to the serverless SQL database using a linked service.
- Passed the view name from the stored procedure script as a parameter.   

## Step 5: Data Reporting
### Connect Power BI with Azure Synapse Analytics

- Opened Power BI and navigated to Get Data -> Azure -> Azure Synapse Analytics (SQL Server).
- Entered the copied serverless SQL endpoint and database name from Azure Synapse.
- Authenticated using a Microsoft account to import data.
- Accessed various views of tables from the SQL database in Synapse.

### Load and Model Data in Power BI

- Imported tables into Power BI and performed data modeling tasks as required.
- Linked relevant tables and created relationships to ensure accurate reporting.

### Create Dashboards

- Designed dashboards using several KPIs (Key Performance Indicators) to visualize and derive insights from the data.
- Integrated dynamic visuals and filters for interactive reporting.

## Security Configuration
- Create Security Group
  - Configured a security group in Azure Active Directory (AAD) to manage access permissions for individuals working within the resource groups.
  - Assigned roles to users to grant or restrict access to specific resources based on their roles and responsibilities.

## Step 6: Testing the End-to-End Pipeline
### Add Schedule Trigger in Azure Data Factory

- Configured a Schedule Trigger in Data Factory to automate the pipeline execution.
- Set up the trigger with appropriate time zones and recurrence intervals to align with business requirements.

### Validate Pipeline Functionality

- Inserted new rows into the original database in SSMS, which is connected to Azure services.
- Verified that the changes in the source database propagated through the pipeline to the Azure Data Lake, Synapse Analytics, and finally to Power BI.
- Confirmed that the new data appeared on the Power BI dashboard after the scheduled trigger executed the pipeline.

<img width="500" alt="pbi" src="https://github.com/user-attachments/assets/5f91a42c-9f5d-40c4-9d97-a19cfafbaece">
<img width="500" alt="pbi1" src="https://github.com/user-attachments/assets/e2bdd896-825b-464f-877d-fb47e976cf19">

  
