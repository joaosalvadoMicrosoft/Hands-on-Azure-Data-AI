# Lab 3: Explore Big Data using Azure Databricks
In this lab you will use Azure Synapse Spark Pools and SQL Serverless Pools to explore data, you will also use Azure Databricks to explore the New York Taxi data files you saved in your data lake in Lab 2. Using a Databricks notebook you will connect to the data lake and query taxi ride details. 

The estimated time to complete this lab is: **45 minutes**.

## Microsoft Learn & Technical Documentation

The following Azure services will be used in this lab. If you need further training resources or access to technical documentation please find in the table below links to Microsoft Learn and to each service's Technical Documentation.

Azure Service | Microsoft Learn | Technical Documentation|
--------------|-----------------|------------------------|
Azure Databricks | [Perform data engineering with Azure Databricks](https://docs.microsoft.com/en-us/learn/paths/data-engineering-with-databricks/) | [Azure Databricks Technical Documentation](https://docs.microsoft.com/en-us/azure/azure-databricks/)
Azure Data Lake Gen2 | [Large Scale Data Processing with Azure Data Lake Storage Gen2](https://docs.microsoft.com/en-us/learn/paths/data-processing-with-azure-adls/) | [Azure Data Lake Gen2 Technical Documentation](https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)
Azure Synapse Analytics | [Implement a Data Warehouse with Azure Synapse Analytics](https://docs.microsoft.com/en-us/learn/paths/implement-sql-data-warehouse/) | [Azure Synapse Analytics Technical Documentation](https://docs.microsoft.com/en-us/azure/sql-data-warehouse/)

## Lab Architecture
![Lab Architecture](./Media/Lab3-Image51.png)


Step     | Description
-------- | -----
![](./Media/Red1.png) |You will use  an Azure Synapse Spark Pools and SQL Serverless Pools to explore data.
![](./Media/Red2.png) |Build an Azure Databricks notebook to explore the data files you saved in your data lake in the previous exercise. You will use Python and SQL commands to open a connection to your data lake and query data from data files. Finally you will integrate datasets from Azure Synapse Analytics data warehouse to your big data processing pipeline. Databricks becomes the bridge between your relational and non-relational data stores.

**IMPORTANT**: Some of the Azure services provisioned require globally unique name and a “-suffix” has been added to their names to ensure this uniqueness. Please take note of the suffix generated as you will need it for the following resources in this lab:

Name	                     |Type
-----------------------------|--------------------
syndtlake*suffix*	         | Data Lake Storage Gen2
asaworkspace*suffix*         | Azure Synapse Analytics workspace
SynapseDW                    | Synapse Dedicated SQL Pool
SparkPool01                  | Apache Spark pool




## Explore Data with Azure Synapse Analytics


Understanding data through data exploration is one of the core challenges faced today by data engineers and data scientists. Depending on the underlying structure of the data as well as the specific requirements of the exploration process, different data processing engines will offer varying degrees of performance, complexity, and flexibility.

In Azure Synapse Analytics, you have the possibility of using either the Synapse SQL Serverless engine, the big-data Spark engine, or both.

In this exercise, you will explore the data lake using both options.

### Create Linked Service connection ADLS Gen 2 and give permissions for reading


1.	First lets create an ADLS Gen 2 connection to our Data Lake. In Azure Synapse studio click in the **Manage option *(toolcase icon)*** on the left-hand side panel. Under **Linked services** menu item, click **+ New** to create a new linked service connection.

    ![](./Media/Lab3-Image26.png)

2.	On the **New Linked Service** blade, type Azure Data Lake Storage Gen2 in the search box to find the **Azure Data Lake Storage Gen2** linked service. Click **Continue**.

    ![](./Media/Lab3-Image27.png)

3.	On the **New Linked Service (Azure Data Lake Storage Gen2)** blade, enter the following details:
    <br>- **Name**: DataLakeQueryConnection
    <br>- **Connect via integration runtime**: AutoResolveIntegrationRuntime
    <br>- **Authentication method**: Account key
    <br>- **Account selection method**: From Azure subscription
    <br>- **Azure subscription**: *[your subscription]*
    <br>- **Storage account name**: syndtlake*suffix*
    
4.	Click **Test connection** to make sure you entered the correct connection details and then click **Create**.

    ![](./Media/Lab3-Image28.png)

5. In the Azure portal go to your Data Lake syndtlake*suffix*, click on **Access Control (IAM)**, then click on **Add** and finally **Add role assignment**, we will give permission to our user to read this data from within synapse SQL and Spark Pools.

    ![](./Media/Lab3-Image29.png)

6. On the **Add role assignment** blade. enter the following details:
    <br>- **Role**: Storage Blob Data Contributor
    <br>- **Assign access to**: User, group, or service principal
    <br>- **Select**: the email of the user who will read data in synapse

7. Click on **Save** button.

    ![](./Media/Lab3-Image41.png)



### Query Data using Synapse SQL Serverless

1.	Connect to your Azure Synapse studio and select the **Data *(Databse icon)*** option.

2.	From the **Data** blade, select the **Linked** tab.

3. Expand **Azure Data Lake Storage Gen2**. Expand the DataLakeQueryConnection account and select nyctaxidata-raw, right click on the last one **yellow_tripdata_2019-06.csv** and on the **New SQL script** choose the **Select TOP 100 rows**. 

    ![](./Media/Lab3-Image31.png)


4. Edit the query to include the header as bellow. Remember to change the name to your data lake.

```sql
SELECT
    TOP 100 *
FROM
    OPENROWSET(
        BULK 'https://syndtlakekpq2soskyd5b4.dfs.core.windows.net/nyctaxidata-raw/yellow_tripdata_2019-01.csv',
        FORMAT = 'CSV',
        HEADER_ROW = TRUE,
        PARSER_VERSION='2.0'
    ) AS [result]
```

5. Notice that you are using a SQL serverless pool and observe the results

    ![](./Media/Lab3-Image32.png)

6. You can explore the Data in the Data Lake using the SQL serverless pools, for example you can count how many records are contained for the CSV files in this container, add the query bellow:

```sql
SELECT
    COUNT_BIG(*)
FROM
    OPENROWSET(
        BULK 'https://syndtlakekpq2soskyd5b4.dfs.core.windows.net/nyctaxidata-raw/*.csv',
        FORMAT = 'CSV',
        HEADER_ROW = TRUE,
        PARSER_VERSION='2.0'
    ) AS [result]

```

> Notice how we updated the path to include all CSV files in folder of `nyctaxidata-raw`.

The output should be **44459136** records.


### Query Data using Synapse Spark Pools

1. Similarly as you have done before, connect to your Azure Synapse studio and select the **Data *(Databse icon)*** option and then select the **Linked** tab.

2. Expand **Azure Data Lake Storage Gen2**. Expand the DataLakeQueryConnection account and select nyctaxidata-raw, right click on the last one **yellow_tripdata_2019-06.csv** and on the **New notebook** select **Load to DataFrame**.

    ![](./Media/Lab3-Image33.png)


3. On the **Attach to** select the SparkPool01, uncomment the ##,header=True .


4. Select **Run all** on the notebook toolbar to execute the notebook, observe the results bellow.


    ![](./Media/Lab3-Image34.png)



    > **Note:** The first time you run a notebook in a Spark pool, Synapse creates a new session. This can take approximately 5 minutes.
    
    > **Note:** To run just the cell, either hover over the cell and select the _Run cell_ icon to the left of the cell, or select the cell then type **Ctrl+Enter** on your keyboard.

    

5. Create a new cell underneath by selecting **{} Add code** when hovering over the blank space at the bottom of the notebook.

    ![](./Media/Lab3-Image35.png)

6. The Spark engine can analyze the CSV files and infer the schema. To do this, enter the following in the new cell:

    ```python
    df.printSchema()
    ```

    Your output should look like the following:

    ```text            
        root
        |-- VendorID: string (nullable = true)
        |-- tpep_pickup_datetime: string (nullable = true)
        |-- tpep_dropoff_datetime: string (nullable = true)
        |-- passenger_count: string (nullable = true)
        |-- trip_distance: string (nullable = true)
        |-- RatecodeID: string (nullable = true)
        |-- store_and_fwd_flag: string (nullable = true)
        |-- PULocationID: string (nullable = true)
        |-- DOLocationID: string (nullable = true)
        |-- payment_type: string (nullable = true)
        |-- fare_amount: string (nullable = true)
        |-- extra: string (nullable = true)
        |-- mta_tax: string (nullable = true)
        |-- tip_amount: string (nullable = true)
        |-- tolls_amount: string (nullable = true)
        |-- improvement_surcharge: string (nullable = true)
        |-- total_amount: string (nullable = true)
        |-- congestion_surcharge: string (nullable = true)
    ```
7. Stop this session by clicking on the **Stop session** followed by **Stop now**

    ![](./Media/Lab3-Image38.png)


8. Again expand the DataLakeQueryConnection account and select nyctaxidata-raw, right click on the last one **yellow_tripdata_2019-06.csv** and on the **New notebook**  this time select **New Spark table**.

    ![](./Media/Lab3-Image36.png)

9. Again, attach to your **SparkPool01** and uncomment the ##,header=True and change the **YourTableName** to **TabelYellow201906** :

    ```python
    %%pyspark
    df = spark.read.load('abfss://nyctaxidata-raw@syndtlakekpq2soskyd5b4.dfs.core.windows.net/yellow_tripdata_2019-06.csv', format='csv'
    ## If header exists uncomment line below
    , header=True
    )
    df.write.mode("overwrite").saveAsTable("default.TabelYellow201906")
    ```

    ![](./Media/Lab3-Image37.png)

10. The command that you ran before created a Spark table, if you go to **Data** blade and then select **Workspace**, expand the **default(Spark)** database and then bellow tables you should be able to see the table you have just created.

    ![](./Media/Lab3-Image40.png)


## Explore Data with Azure Databricks


**IMPORTANT**: The code snippets below illustrate the simplest and quickest way to establish connections between Databricks and other Azure services. They **ARE NOT** considered best practices as they expose secrets and passwords in plain text. For a secure implementation following the security best practices, please consider the use of Azure Key Vault in conjuntion with Databricks Secret Scopes (https://docs.azuredatabricks.net/user-guide/secrets/secret-scopes.html).

### Create Azure Databricks Cluster 
In this section you are going to create an Azure Databricks cluster that will be used to execute notebooks.


1.	In the Azure Portal, navigate to the lab resource group and locate the Azure Databricks resource ADPDatabricks-*suffix*.

2.	On the **ADPDatabricks-*suffix*** blade, click the **Launch Workspace** button. The Azure Databricks portal will open on a new browser tab.

    ![](./Media/Lab3-Image42.png)

3.	On the Azure Databricks portal, click the **Clusters** button on the left-hand side menu. 

4.	On the **Clusters** blade, click **+ Create Cluster**.

    ![](./Media/Lab3-Image03.png)

5.	On the **Create Cluster** blade, enter the following connection details:
    <br>- **Cluster Name**: ADPDatabricksCluster
    <br>- **Terminate after**: 120 minutes
    <br>- **Min Workers**: 0
    <br>- **Max Workers**: 4

    Leave all other fields with their default values.

6.	Click **Create Cluster**. It should take around 5 to 7 minutes for the cluster to be fully operational. Observe the different Worker Types that you have available to use.

    ![](./Media/Lab3-Image43.png)

### Create an Azure Databricks Notebook 
In this section you are going to create an Azure Databricks notebook that will be used to explore the taxi data files you copied to your data lake in the Lab 2. 


1.	On the Azure Databricks portal, click the **Home** button on the left-hand side menu. 

2.	On the **Workspace** blade, click the down arrow next to your user name and then click **Create > Notebook**.

    ![](./Media/Lab3-Image05.jpg)

3.	On the **Create Notebook** pop-up window type “NYCTaxiData” in the Name field.

4.	Ensure you have the **Language** field set to **Python** and the **Cluster** field is set to **ADPDatabricksCluster**.

5.	Click **Create**.

    ![](./Media/Lab3-Image06.png)

6.	On the **Cmd 1** cell, you will invoke the Spark API to establish a connection to your SynapseDataLake storage account. For this you will need to retrieve the name and the key of your SynapseDataLake storage account from the Azure Portal. 

    ![](./Media/Lab3-Image44.png)

7.	Use the Python code below and replace *synapsedatalake[suffix]* with your own suffix on line 22 and to replace *[your synapsedatalake account key]* with the data lake account key on line 23.


```python
#Setup connection to SynapseDataLake storage account

def MountDataLakeContainer(datalakeAccountName, datalakeAccountKey, containerName, mountPointName):
    dataLakeAccountDNS = datalakeAccountName + ".blob.core.windows.net"
    
    dataLakeAuthTypeConfig = "fs.azure.account.auth.type." + dataLakeAccountDNS
    dataLakeKeyConfig = "fs.azure.account.key." + dataLakeAccountDNS
    
    dataLakeExtraConfig = {dataLakeAuthTypeConfig:"SharedKey"
                           , dataLakeKeyConfig:datalakeAccountKey}
    
    containerUri = "wasbs://" + containerName + "@" + dataLakeAccountDNS
    mountPointFolder = "/mnt/" + mountPointName

    try:
        dbutils.fs.mount(source = containerUri, mount_point = mountPointFolder, extra_configs = dataLakeExtraConfig)
    except Exception as e:
        if "Directory already mounted" in str(e):
            pass # Ignore error if already mounted.
        else:
            raise e
    return containerUri

#Set Account Name and Key Variable Values
dataLakeAccountName = 'syndtlake[suffix]' #<-- Your syndtlake account name goes here. Remember to replace <suffix> with your own suffix.
dataLakeAccountKey = '[your synapsedatalake account key]' #<-- Your syndtlake account key goes here.

#Mount NYCTaxiData-Raw and NYCTaxiData-Curated Containers
MountDataLakeContainer(dataLakeAccountName, dataLakeAccountKey, 'nyctaxidata-raw', 'raw')
MountDataLakeContainer(dataLakeAccountName, dataLakeAccountKey, 'nyctaxidata-curated', 'curated')
```
8.	Press **Shift + Enter** to execute and create a new notebook cell. 

9.	In the **Cmd 2** cell, define a new **StructType** object that will contain the definition of the data frame schema.

10.	Using the schema defined in the previous step, initialise a new data frame by invoking the Spark API to read the contents of the nyctaxidata-raw container in the SynapseDataLake storage account. Use the Python code below:

```python
#Define NYCTaxiData schema and load data into a Data Frame

from pyspark.sql.types import *

nycTaxiDataSchema = StructType([
  StructField("VendorID",IntegerType(),True)
  , StructField("tpep_pickup_datetime",DateType(),True)
  , StructField("tpep_dropoff_datetime",DateType(),True)
  , StructField("passenger_count",IntegerType(),True)
  , StructField("trip_distance",DoubleType(),True)
  , StructField("RatecodeID",IntegerType(),True)
  , StructField("store_and_fwd_flag",StringType(),True)
  , StructField("PULocationID",IntegerType(),True)
  , StructField("DOLocationID",IntegerType(),True)
  , StructField("payment_type",IntegerType(),True)
  , StructField("fare_amount",DoubleType(),True)
  , StructField("extra",DoubleType(),True)
  , StructField("mta_tax",DoubleType(),True)
  , StructField("tip_amount",DoubleType(),True)
  , StructField("tolls_amount",DoubleType(),True)
  , StructField("improvement_surcharge",DoubleType(),True)
  , StructField("total_amount",DoubleType(),True)])
  
dfNYCTaxiData = spark.read.format('csv').options(header='true').schema(nycTaxiDataSchema).load('/mnt/raw/*.csv')
dfNYCTaxiData.cache()
```

11.	Hit **Shift + Enter** to execute the command and create a new cell. 

12.	In the **Cmd 3** cell, call the display function to show the contents of the data frame dfNYCTaxiData. Use the Python code below:

```python
#Display Data Frame Content

display(dfNYCTaxiData)
```
13.	Hit **Shift + Enter** to execute the command and create a new cell. You will see a data grid showing the top 1000 records from the dataframe:

    ![](./Media/Lab3-Image10.png)

14.	In the **Cmd 4** cell, call the **select()** method of the data frame object to select the columns "tpep_pickup_datetime", "passenger_count" and "total_amount". Then use the  **filter()** method to filter rows where "passenger_count > 6" and "total_amount > 50.0". Use the Python code below:

```python
# Use Data Frame API Operations to Filter Data

display(dfNYCTaxiData.select("tpep_pickup_datetime", "passenger_count", "total_amount").filter("passenger_count > 6 and total_amount > 50.0"))
```

15.	Hit **Shift + Enter** to execute the command and create a new cell. 

    ![](./Media/Lab3-Image16.png)

16.	In the **Cmd 5** cell, call the **createOrReplaceTempView** method of the data frame object to create a temporary view of the data in memory. Use the Python code below:

```python
# Create Local Temp View

dfNYCTaxiData.createOrReplaceTempView('NYCTaxiDataTable')
```
17.	Hit **Shift + Enter** to execute the command and create a new cell. 

18.	In the **Cmd 6** cell, change the cell language to SQL using the *%sql* command. 

19.	Write a SQL query to retrieve the total number of records in the NYCTaxiDataTable view. Use the command below:

```sql
%sql
--Use SQL to count NYC Taxi Data records

select count(*) from NYCTaxiDataTable
```

20.	Hit **Shift + Enter** to execute the command and create a new cell. You will see the total number of records in the data frame at the bottom of the cell.

    ![](./Media/Lab3-Image11.png)

21.	In the **Cmd 7** cell, write a SQL query to filter taxi rides that happened on the Apr, 7th 2019 that had more than 5 passengers. Use the command below:

```sql
%sql
-- Use SQL to filter NYC Taxi Data records

select cast(tpep_pickup_datetime as date) as pickup_date
  , tpep_dropoff_datetime
  , passenger_count
  , total_amount
from NYCTaxiDataTable
where cast(tpep_pickup_datetime as date) = '2019-04-07'
  and passenger_count > 5
```

22.	Hit **Shift + Enter** to execute the command and create a new cell. You will see a grid showing the filtered result set.

    ![](./Media/Lab3-Image12.png)

23.	In the **Cmd 8** cell, write a SQL query to aggregate records and return total number of rides by payment type. Use the command below:

```sql
%sql
-- Use SQL to aggregate NYC Taxi Data records and visualize data
select case payment_type
            when 1 then 'Credit card'
            when 2 then 'Cash'
            when 3 then 'No charge'
            when 4 then 'Dispute'
            when 5 then 'Unknown'
            when 6 then 'Voided trip'
        end as PaymentType
  , count(*) as TotalRideCount
from NYCTaxiDataTable
group by payment_type
order by TotalRideCount desc

```

24.	Hit **Shift + Enter** to execute the command and create a new cell. Results will be displayed in a grid in the cell.

25.	Click the **Bar chart** button to see results as a bar chart.

    ![](./Media/Lab3-Image13.png)
    ![](./Media/Lab3-Image14.png)


26. Using Python, open a JDBC connection to your Azure Synapse Analytics and load Taxi location lookup data from the Staging.NYCTaxiLocationLookup table into a new data frame called dfLocationLookup. 

    **IMPORTANT**: Don't forget to replace the 'asaworkspace-*suffix*' with your specific Azure Synapse Analytics server name.

    In the same cell, create a temporary view called "NYCTaxiLocation" and display the contents of the data frame. Use the Python code below:

```python
# Load Taxi Location Data from Azure Synapse Analytics

jdbcUrl = "jdbc:sqlserver://asaworkspace-suffix.sql.azuresynapse.net:1433;database=SynapseDW" #Replace "suffix" with your own  
connectionProperties = {
  "user" : "adpadmin",
  "password" : "P@ssw0rd123!",
  "driver" : "com.microsoft.sqlserver.jdbc.SQLServerDriver"
}

pushdown_query = '(select * from NYC.TaxiLocationLookup) as t'
dfLookupLocation = spark.read.jdbc(url=jdbcUrl, table=pushdown_query, properties=connectionProperties)

dfLookupLocation.createOrReplaceTempView('NYCTaxiLocation')

display(dfLookupLocation) 

```

27. Hit **Shift + Enter** to execute the command and create a new cell. Results will be displayed in a grid in the cell.

    ![](./Media/Lab3-Image45.png)

28. In the **Cmd 10** cell, write a SQL query to cleanse data and apply standard column definitions. You should joining the two dataframes using their view names and filter out any invalid data. The resulting dataset should be saved in a Spark table using Parquet files sitting in the NYCTaxiData-Curated container in your SynapseDataLake storage account. Use the SQL command below:

```sql
%sql
--Create Spark table with cleansed, validated dataset combining raw CSV files and reference data from you Synapse data warehouse
--Data will be saved into the NYCTaxiData-Curated container in your SynapseDataLake account

create table if not exists NYCTaxiData_Curated
using parquet partitioned by (PickUpYearMonth) --Use Parquet format partitioned by YYYY-MM
options ('compression'='snappy') --Parquet compression options
location '/mnt/curated/' --Data to be saved in the NYCTaxiData-Curated container in your SynapseDataLake storage account.
as
select 
    VendorID
    , concat(year(tpep_pickup_datetime), '-', format_string('%02d',month(tpep_pickup_datetime),'##')) as PickUpYearMonth --Partition Key
    , cast(tpep_pickup_datetime as timestamp) as PickUpDateTime
    , cast(tpep_dropoff_datetime as timestamp) as DropOffDateTime
    , passenger_count as PassengerCount
    , trip_distance as TripDistance
    , cast(PULocationID as int) as PickUpLocationID
    , pu.Zone as PickUpLocationZone
    , pu.Borough as PickUpLocationBorough
    , cast(DOLocationID as int) as DropOffLocationID
    , do.Zone as DropOffLocationZone
    , do.Borough as DropOffLocationBorough
    , cast(payment_type as int) as PaymentTypeID
    , case payment_type
            when 1 then 'Credit card'
            when 2 then 'Cash'
            when 3 then 'No charge'
            when 4 then 'Dispute'
            when 5 then 'Unknown'
            when 6 then 'Voided trip'
        end as PaymentTypeDescription
    , cast(case when fare_amount < 0 then 0.00 else fare_amount end as decimal(8,2)) as FareAmount --Cleanse invalid data
    , cast(case when extra < 0 then 0.00 else extra end as decimal(8,2)) as ExtraAmount --Cleanse invalid data
    , cast(case when mta_tax < 0 then 0.00 else mta_tax end as decimal(8,2)) as MTATaxAmount --Cleanse invalid data
    , cast(case when tip_amount < 0 then 0.00 else tip_amount end as decimal(8,2)) as TipAmount --Cleanse invalid data
    , cast(case when tolls_amount < 0 then 0.00 else tolls_amount end as decimal(8,2)) as TollsAmount --Cleanse invalid data
    , cast(case when improvement_surcharge < 0 then 0.00 else improvement_surcharge end as decimal(8,2)) as ImprovementSurchargeAmount --Cleanse invalid data
    , cast(case when total_amount < 0 then 0.00 else total_amount end as decimal(8,2)) as TotalRideAmount --Cleanse invalid data
from NYCTaxiDataTable as rides
  join NYCTaxiLocation as pu
    on rides.PULocationID = pu.LocationID
  join NYCTaxiLocation as do
    on rides.DOLocationID = do.LocationID
where passenger_count > 0 --Data Cleanup Rules
  and year(tpep_pickup_datetime) = 2019
```

39. Hit **Shift + Enter** to execute the command. Results will be displayed in a grid in the cell.

40. In the **Cmd 11** cell, write a SQL query to retrieve the first 100 rows from the Spark table NYCTaxiData_Curated you just created in the previous step. Use the SQL command below.

```sql
%sql

--Retrieve curated dataset sample

select *
from NYCTaxiData_Curated
limit 100
```
![](./Media/Lab3-Image50.png)

### Review the NYCTaxiData-Curated Container Content
In this section you are going to review the content of the NYCTaxiData-Curated container in your syndtlake storage account after you created the Spark table using the Databricks notebook in the previous exercise. You should see a list of folders created, one for each partition and inside them the collection of Parquet files containing cleansed and validated taxi data ready to be used for reporting.



1.	In the Azure Portal, go to the lab resource group and locate the Azure Storage account **syndtlake*suffix***. 
2.	On the **Overview** panel, click **Containers**.

    ![](./Media/Lab3-Image46.png)

3. Open the NYCTaxiData-Curated container to see its contents. You should see a folder structure similar to this one:

    ![](./Media/Lab3-Image47.png)

4. Inside each folder you will find the collection of individual Parquet files that make up the NYCTaxiData_Curated Spark table.

    ![](./Media/Lab3-Image48.png)