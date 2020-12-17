# Lab 2: Transform Big Data using Azure Synapse Pipelines Mapping Data Flows
In this lab you will use Azure Synapse Pipelines to download large data files to your data lake and use Mapping Dataflows to generate a summary dataset and store it. The dataset you will use contains detailed New York City Yellow Taxi rides for the first half of 2019. You will generate a daily aggregated summary of all rides using Mapping Data Flows and save the resulting dataset in your Azure Synapse Analytics. Finally you will use Power BI to visualise summarised taxi ride data.

The estimated time to complete this lab is: **60 minutes**.

## Microsoft Learn & Technical Documentation

The following Azure services will be used in this lab. If you need further training resources or access to technical documentation please find in the table below links to Microsoft Learn and to each service's Technical Documentation.

Azure Service | Microsoft Learn | Technical Documentation|
--------------|-----------------|------------------------|
Azure Data Lake Gen2 | [Large Scale Data Processing with Azure Data Lake Storage Gen2](https://docs.microsoft.com/en-us/learn/paths/data-processing-with-azure-adls/) | [Azure Data Lake Gen2 Technical Documentation](https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)
Azure Synapse Analytics | [Implement a Data Warehouse with Azure Synapse Analytics](https://docs.microsoft.com/en-us/learn/paths/implement-sql-data-warehouse/) | [Azure Synapse Analytics Technical Documentation](https://docs.microsoft.com/en-us/azure/sql-data-warehouse/)

## Lab Architecture
![Lab Architecture](./Media/Lab2-Image128.png)

Step     | Description
-------- | -----
![](./Media/Green1.png) | Build an Azure Synapse Pipeline to copy big data files from shared Azure Storage 
![](./Media/Green2.png) | Ingest Files in your Data Lake
![](./Media/Green3.png) | Use Mapping Data Flows to generate a aggregated daily summary and save the resulting dataset into your Azure Synapse Analytics data warehouse.
![](./Media/Green4.png) | Visualize data from your Azure Synapse Analytics using Power BI

**IMPORTANT**: Some of the Azure services provisioned require globally unique name and a “-suffix” has been added to their names to ensure this uniqueness. Please take note of the suffix generated as you will need it for the following resources in this lab:



Name	                     |Type
-----------------------------|--------------------
syndtlake*suffix*	         | Data Lake Storage Gen2
asaworkspace*suffix*         | Azure Synapse Analytics workspace
SynapseDW                    | Synapse Dedicated SQL Pool


## Create Azure Synapse Analytics database objects
In this section you will connect to Azure Synapse Analytics Studio to create the data warehouse objects used to host and process data.



1.	As you have done previously connect to your Azure Synapse studio and select the **Data *(Databse icon)*** option:

    ![](./Media/Lab1-Image101.png)

2.	Expand the the Databases, click on the **...** and then selecte "Empty script":

    ![](./Media/Lab1-Image102.png)

7.	Copy the code bellow and run it, this will create two new round robin distributed tables named [NYC].[TaxiDataSummary] and [NYC].[TaxiLocationLookup]. Use the script below:

    ```sql
    create table [NYC].[TaxiDataSummary]
    (
        [PickUpDate] [date] NULL,
        [PickUpBorough] [varchar](200) NULL,
        [PickUpZone] [varchar](200) NULL,
        [PaymentType] [varchar](11) NULL,
        [TotalTripCount] [int] NULL,
        [TotalPassengerCount] [int] NULL,
        [TotalDistanceTravelled] [decimal](38, 2) NULL,
        [TotalTipAmount] [decimal](38, 2) NULL,
        [TotalFareAmount] [decimal](38, 2) NULL,
        [TotalTripAmount] [decimal](38, 2) NULL
    )
    with
    (
        distribution = round_robin,
        clustered columnstore index
    )

    go

    create table [NYC].[TaxiLocationLookup]
    (
        [LocationID] [int] NULL,
        [Borough] [varchar](200) NULL,
        [Zone] [varchar](200) NULL,
        [ServiceZone] [varchar](200) NULL
    )
    with
    (
        distribution = round_robin,
        clustered columnstore index
    )
    go
    ```

8. You should observe now 2 new Tables created.

    ![](./Media/Lab2-Image129.png)



## Create Linked Service connection to MDWResources
In this section you will create a linked service connection to a shared storage accounnt called MDWResources hosted in an external Azure subscription. This storage account hosts the NYC Taxi data files you will copy to your data lake. As this storage account sits in an external subscription you will connect to it using a SAS URL token.


1.	In Azure Synapse studio click in the **Manage option *(toolcase icon)*** on the left-hand side panel. Under **Linked services** menu item, click **+ New** to create a new linked service connection.

    ![](./Media/Lab2-Image79.png)

2.	On the **New Linked Service** blade, type “Azure Blob Storage” in the search box to find the **Azure Blob Storage** linked service. Click **Continue**.

    ![](./Media/Lab2-Image10.png)

3.	On the **New Linked Service (Azure Blob Storage)** blade, enter the following details:
    <br>- **Name**: MDWResources
    <br>- **Connect via integration runtime**: AutoResolveIntegrationRuntime
    <br>- **Authentication method**: SAS URI
    <br>- **SAS URL**: 
    ```
    https://mdwresources.blob.core.windows.net/?sv=2018-03-28&ss=b&srt=sco&sp=rwl&se=2050-12-30T17:25:52Z&st=2019-04-05T09:25:52Z&spr=https&sig=4qrD8NmhaSmRFu2gKja67ayohfIDEQH3LdVMa2Utykc%3D
    ```
4.	Click **Test connection** to make sure you entered the correct connection details and then click **Finish**.

    ![](./Media/Lab2-Image80.png)

## Create Source and Destination Data Sets
In this section you are going to create 6 datasets that will be used by your data pipeline:

Dataset |Role           |Linked Service| Description|
--------|---------------|----------------|---------------
**MDWResources_NYCTaxiData_Binary**| Source |MDWResources|References MDWResources shared storage account container that contains source NYC Taxi data files.
**SynapseDataLake_NYCTaxiData_Binary**| Destination |synapsedatalake|References your synapsedatalake-*suffix* storage account. It acts as the destination for the NYC Taxi data files copied from MDWResources_NYCTaxiData_Binary.
**NYCDataSets_NYCTaxiLocationLookup**| Source |OperationalSQL_NYCDataSets|References [NYC].[TaxiLocationLookup] table on the NYCDataSets database. This table contains records with all taxi location codes and names.
**SynapseDW_NYCTaxiLocationLookup**| Destination|SynapseSQL_SynapseDW|References the destination table [NYC].[TaxiLocationLookup] in the Azure Synapse Analytics data warehouse SynapseDW and acts as destination of lookup data copied from NYCDataSets_NYCTaxiLookup.
**SynapseDataLake_NYCTaxiData_CSV**| Source |synapsedatalake| References the NYCTaxiData-Raw container in your SynapseDataLake-*suffix* storage account. It functions as a data source for the Mapping Data Flow.
**SynapseDW_NYCTaxiDataSummary**|SynapseSQL_SynapseDW| Destination | References the table [NYC].[TaxiDataSummary] in the Azure Synapse Analytics and acts as the destination for the summary data generated by your Mapping Data Flow.



1.	In the **Azure Synapse Studio** click in the **Data *(Databse icon)*** option on the left-hand side panel. Click on the "+" and then select **Integration dataset** to create a new dataset.

    ![](./Media/Lab2-Image81.png)

2.	Type “Azure Blob Storage” in the search box and select **Azure Blob Storage**. Click **Continue**.

    ![](./Media/Lab2-Image13.png)

3.	On the **Select Format** blade, select **Binary** and click **Continue**.

    ![](./Media/Lab2-Image14.png)

4.	On the **Set Properties** blade, enter the following details:
    <br>- **Name**: MDWResources_NYCTaxiData_Binary
    <br>- **Linked service**: MDWResources
    <br>- **File Path**: **Container**: nyctaxidata, **Directory**: [blank], **File**: [blank]
    
    ![](./Media/Lab2-Image82.png)

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "MDWResources_NYCTaxiData_Binary",
        "properties": {
            "linkedServiceName": {
                "referenceName": "MDWResources",
                "type": "LinkedServiceReference"
            },
            "folder": {
                "name": "Lab2"
            },
            "annotations": [],
            "type": "Binary",
            "typeProperties": {
                "location": {
                    "type": "AzureBlobStorageLocation",
                    "container": "nyctaxidata"
                }
            }
        }
    }
    ```
5.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image15.png)

6.	Repeat the process to create another Azure Storage Binary dataset, this time referencing the NYCTaxiData container in your SynapseDataLake storage account. This dataset acts as the destination for the NYC taxi data files you will copy from the previous dataset.

7.	Type “Azure Blob Storage” in the search box and select **Azure Blob Storage**. Click **Continue**.

    ![](./Media/Lab2-Image13.png)

8.	On the **Select Format** blade, select **Binary** and click **Continue**.

    ![](./Media/Lab2-Image14.png)

9.	On the **Set Properties** blade, enter the following details:
    <br>- **Name**: SynapseDataLake_NYCTaxiData_Binary
    <br>- **Linked Service**: SynapseDataLake
    <br>- **File Path**: **Container**: nyctaxidata-raw, **Directory**: [blank], **File**: [blank]

    ![](./Media/Lab2-Image84.png)

    Click **Continue**.

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "SynapseDataLake_NYCTaxiData_Binary",
        "properties": {
            "linkedServiceName": {
                "referenceName": "SynapseDataLake",
                "type": "LinkedServiceReference"
            },
            "folder": {
                "name": "Lab2"
            },
            "annotations": [],
            "type": "Binary",
            "typeProperties": {
                "location": {
                    "type": "AzureBlobStorageLocation",
                    "container": "nyctaxidata-raw"
                }
            }
        }
    }
    ```
10.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image85.png)

11.	Repeat the process to create a new Azure Storage CSV dataset referencing the NYCTaxiData container in your SynapseDataLake storage account. This dataset acts as the data source of NYC taxi records (CSV) you will copy to your Azure Synapse Analytics.

    **IMPORTANT**: You will need to download the sample file from https://aka.ms/TaxiDataSampleFile to your Desktop. This file will be used to derive the schema for the dataset. 
    
    The reason you need this step is because you will need to work with column names in the mapping data flow, but at this time you don't have the data files in your data lake yet.

12.	Type “Azure Blob Storage” in the search box and select **Azure Blob Storage**. Click **Continue**.

    ![](./Media/Lab2-Image13.png)

13.	On the **Select Format** blade, select **DelimitedText** and click **Continue**.

    ![](./Media/Lab2-Image43.png)

14.	On the **Set Properties** blade, enter the following details:
    <br>- **Name**: SynapseDataLake_NYCTaxiData_CSV
    <br>- **Linked Service**: SynapseDataLake
    <br>- **File Path**: **Container**: nyctaxidata-raw, **Directory**: [blank], **File Path**: [blank]
    <br>- **First row as header**: Checked
    <br>- **Import schema**: From sample file > [select the sample file you downloaded in step 11]

    ![](./Media/Lab2-Image86.png)

    Click **OK**.

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "SynapseDataLake_NYCTaxiData_CSV",
        "properties": {
            "linkedServiceName": {
                "referenceName": "SynapseDataLake",
                "type": "LinkedServiceReference"
            },
            "annotations": [],
            "type": "DelimitedText",
            "typeProperties": {
                "location": {
                    "type": "AzureBlobStorageLocation",
                    "container": "nyctaxidata-raw"
                },
                "columnDelimiter": ",",
                "escapeChar": "\\",
                "firstRowAsHeader": true,
                "quoteChar": "\""
            },
            "schema": [
                {
                    "name": "VendorID",
                    "type": "String"
                },
                {
                    "name": "tpep_pickup_datetime",
                    "type": "String"
                },
                {
                    "name": "tpep_dropoff_datetime",
                    "type": "String"
                },
                {
                    "name": "passenger_count",
                    "type": "String"
                },
                {
                    "name": "trip_distance",
                    "type": "String"
                },
                {
                    "name": "RatecodeID",
                    "type": "String"
                },
                {
                    "name": "store_and_fwd_flag",
                    "type": "String"
                },
                {
                    "name": "PULocationID",
                    "type": "String"
                },
                {
                    "name": "DOLocationID",
                    "type": "String"
                },
                {
                    "name": "payment_type",
                    "type": "String"
                },
                {
                    "name": "fare_amount",
                    "type": "String"
                },
                {
                    "name": "extra",
                    "type": "String"
                },
                {
                    "name": "mta_tax",
                    "type": "String"
                },
                {
                    "name": "tip_amount",
                    "type": "String"
                },
                {
                    "name": "tolls_amount",
                    "type": "String"
                },
                {
                    "name": "improvement_surcharge",
                    "type": "String"
                },
                {
                    "name": "total_amount",
                    "type": "String"
                },
                {
                    "name": "congestion_surcharge",
                    "type": "String"
                }
            ]
        }
    }
    ```
15.	Repeat the process to create an Azure SQL Database dataset. It references the NYC.TaxiLocationLookup table in the NYCDataSets database.

16.	Type “Azure SQL Database" in the search box and select **Azure SQL Database**. Click **Continue**.

    ![](./Media/Lab1-Image69.png)

17.	On the **Set Properties** blade, enter the following details:
    <br>- **Name**: NYCDataSets_NYCTaxiLocationLookup
    <br>- **Linked Service**: OperationalSQL_NYCDataSets
    <br>- **Table name**: NYC.TaxiLocationLookup
    <br>- **Import schema**: From connection/store.

    ![](./Media/Lab2-Image87.png)

18.	Leave remaining fields with default values.  

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "NYCDataSets_NYCTaxiLocationLookup",
        "properties": {
            "linkedServiceName": {
                "referenceName": "OperationalSQL_NYCDataSets",
                "type": "LinkedServiceReference"
            },
            "folder": {
                "name": "Lab2"
            },
            "annotations": [],
            "type": "AzureSqlTable",
            "schema": [],
            "typeProperties": {
                "schema": "NYC",
                "table": "TaxiLocationLookup"
            }
        }
    }
    ```

19.	Repeat the process to create another dataset, this time referencing the NYC.TaxiDataSummary in your Azure Synapse Analytics database. 

20.	Type “Azure Synapse Analytics” in the search box and select **Azure Synapse Analytics**. Click **Continue**.

    ![](./Media/Lab1-Image171.png)

21.	On the Set Properties blade, enter the following details:
    <br>- **Name**: SynapseDW_NYCTaxiDataSummary
    <br>- **Linked Service**: SynapseSQL_SynapseDW
    <br>- **Table**: [NYC].[TaxiDataSummary]
    <br>- **Import schema**: From connection/store

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "SynapseDW_NYCTaxiDataSummary",
        "properties": {
            "linkedServiceName": {
                "referenceName": "SynapseSQL_SynapseDW",
                "type": "LinkedServiceReference"
            },
            "folder": {
                "name": "Lab2"
            },
            "annotations": [],
            "type": "AzureSqlDWTable",
            "schema": [
                {
                    "name": "PickUpDate",
                    "type": "date"
                },
                {
                    "name": "PickUpBorough",
                    "type": "varchar"
                },
                {
                    "name": "PickUpZone",
                    "type": "varchar"
                },
                {
                    "name": "PaymentType",
                    "type": "varchar"
                },
                {
                    "name": "TotalTripCount",
                    "type": "int",
                    "precision": 10
                },
                {
                    "name": "TotalPassengerCount",
                    "type": "int",
                    "precision": 10
                },
                {
                    "name": "TotalDistanceTravelled",
                    "type": "decimal",
                    "precision": 38,
                    "scale": 2
                },
                {
                    "name": "TotalTipAmount",
                    "type": "decimal",
                    "precision": 38,
                    "scale": 2
                },
                {
                    "name": "TotalFareAmount",
                    "type": "decimal",
                    "precision": 38,
                    "scale": 2
                },
                {
                    "name": "TotalTripAmount",
                    "type": "decimal",
                    "precision": 38,
                    "scale": 2
                }
            ],
            "typeProperties": {
                "schema": "NYC",
                "table": "TaxiDataSummary"
            }
        }
    }
    ```

22.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image88.png)

23.	Repeat the process to create another dataset, this time referencing the [NYC].[TaxiLocationLookup] in your Azure Synapse Analytics database. 

24.	Type “Azure Synapse Analytics” in the search box and select **Azure Synapse Analytics**. Click **Continue**.

![](./Media/Lab1-Image171.png)

25.	On the Set Properties blade, enter the following details:
    <br>-**Name**: SynapseDW_NYCTaxiLocationLookup
    <br>-**Linked Service**: SynapseSQL_SynapseDW
    <br>-**Table**: [NYC].[TaxiLocationLookup]
    <br>-**Import schema**: From connection/store

    Alternatively you can copy and paste the Dataset JSON definition below:

    ```json
    {
        "name": "SynapseDW_NYCTaxiLocationLookup",
        "properties": {
            "linkedServiceName": {
                "referenceName": "SynapseSQL_SynapseDW",
                "type": "LinkedServiceReference"
            },
            "folder": {
                "name": "Lab2"
            },
            "annotations": [],
            "type": "AzureSqlDWTable",
            "schema": [
                {
                    "name": "LocationID",
                    "type": "int",
                    "precision": 10
                },
                {
                    "name": "Borough",
                    "type": "varchar"
                },
                {
                    "name": "Zone",
                    "type": "varchar"
                },
                {
                    "name": "service_zone",
                    "type": "varchar"
                }
            ],
            "typeProperties": {
                "schema": "NYC",
                "table": "TaxiLocationLookup"
            }
        }
    }
    ```

26.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image89.png)

8. Under **Linked** tab, click the ellipsis **(…)** next to **Integration datasets** and then click **New folder** to create a new Folder. Name it **Lab2**.

    ![](./Media/Lab2-Image90.png)

9. Drag the two datasets created into the **Lab2** folder you just created.

    ![](./Media/Lab2-Image91.png)

29.	Publish your dataset changes by clicking the **Publish all** button.

    ![](./Media/Lab2-Image92.png)


## Create a Mapping Data Flow Integration Runtime (Optional)
In this section you are going to create an integration runtime for Mapping Data Flow executions.
Mapping data flows are visually designed data transformations in Azure Synapse pipelines and Azure Data Factory. Data flows allow data engineers to develop data transformation logic without writing code. The resulting data flows are executed as activities within pipelines that use scaled-out Apache Spark clusters.
By creating a custom integration runtime you have the option to set the compute configuration for your Spark cluster. 



1. In the **Azure Synapse Studio** click the **Manage *(toolcase icon)*** option on the left-hand side panel. Under **Integration runtimes** tab, click **+ New** to create a new integration runtime.

    ![](./Media/Lab2-Image93.png)

2. On the **Integration runtime setup** blade, select **Azure** and click **Continue**.

    ![](./Media/Lab2-Image94.png)

3. On the next page **Integration runtime setup** blade enter the following details:
    <br>- **Name**: MappingDataFlowsIR
    <br>- **Region**: Auto Resolve
    <br>- **Data Flow runtime > Compute type**: General Purpose
    <br>- **Data Flow runtime > Core count**: 8 (+ 8 Driver cores)

    ![](./Media/Lab2-Image95.png)

5. Click **Create** to create the integration runtime.

## Create a Mapping Data Flow
In this section you are going to create a Mapping Data Flow that will transform the Taxi detailed records into an aggreated daily summary. The Mapping Data Flow will read all records from the files stored in your SynapseDataLake account and apply a sequence of transformations before the aggregated summary can be saved into the NYC.TaxiDataSummary table in your Azure Synapse Analytics.

1. In the **Azure Synapse Analytics** portal click in the **Develop *(pages icon)*** option on the left-hand side panel. Click the  **+** and then click **Data flow** to create a new Data Flow.

    ![](./Media/Lab2-Image96.png)

2.	On the Data Flow properties, enter the following details:
    <br>- **General > Name**: TransformNYCTaxiData
    
3. On the design surface click **Add Source**. On the source properties enter the following details:
    <br>- **Source Settings > Output stream name**: TaxiDataFiles
    <br>- **Source Settings > Source dataset**: SynapseDataLake_NYCTaxiData_CSV

    ![](./Media/Lab2-Image97.png)

4. Repeat the process above and add another data source. On the source properties enter the following details:
    <br>- **Source Settings > Output stream name**: TaxiLocationLookup
    <br>- **Source Settings > Source dataset**: SynapseDW_NYCTaxiLocationLookup

    ![](./Media/Lab2-Image98.png)

5. Click on the **+** sign next to the **TaxiDataFiles** source and type "Derived Column" in the search box. Click the **Derived Column** schema modifier.

    ![](./Media/Lab2-Image99.png)

6. On the Derived Column's properties, enter the following details:
    <br>- **Derived column's settings > Output stream name**: TransformColumns

7. Still on the Derived column's settings, under the **Columns** option add the following column name and expression:
    
    * Name: **PaymentType**
    
    Click the **Enter expression...** text box and enter the following expression in the **Expression for field "PaymentType"**:

    ```
    case (payment_type == '1', 'Credit card'
        , payment_type == '2', 'Cash'
        , payment_type == '3', 'No charge'
        , payment_type == '4', 'Dispute'
        , payment_type == '5', 'Unknown'
        , payment_type == '6', 'Voided trip')
    ```

    ![](./Media/Lab2-Image100.png)


    Click the "+" sign next to the expression for PaymentType to add a new derived column. Click **Add column** from the menu.

    ![](./Media/Lab2-Image53.png)

    Repeat the process to create the following derived columns using the names and expressions below:

    |Column Name |Expression|
    | -----| ----- |
    |PickUpDate | ```toDate(tpep_pickup_datetime,'yyyy-MM-dd')``` |
    |PickUpLocationID|```toInteger(PULocationID)```|
    |PassengerCount|```toInteger(passenger_count)```|
    |DistanceTravelled|```toDecimal(trip_distance)```|
    |TipAmount|```toDecimal(tip_amount)```|
    |FareAmount|```toDecimal(fare_amount)```|
    |TotalAmount|```toDecimal(total_amount)```|

    
    Your full list of derived columns should look like this:

    ![](./Media/Lab2-Image101.png)

8. Click on the **+** sign next to the **TransformColumn** transformation and type "Join" in the search box. Click the **Join** transformation.

    ![](./Media/Lab2-Image56.png)

9. On the Join properties, enter the following details:
    <br>- **Join Settings > Output stream name**: JoinPickUpLocation
    <br>- **Join Settings > Left stream**: TransformColumns
    <br>- **Join Settings > Right stream**: TaxiLocationLookup
    <br>- **Join Settings > Join type**: Inner
    <br>- **Join Settings > Join conditions > Left**: PickUpLocationID
    <br>- **Join Settings > Join conditions > Right**: LocationID

    ![](./Media/Lab2-Image102.png)

10. Click on the **+** sign next to the **JoinPickUpLocation**  transformation and type "Aggregate" in the search box. Click the **Aggregate** schema modifier.

    ![](./Media/Lab2-Image58.png)

11. On the Aggregate properties, enter the following details:
    <br>- **Aggregate Settings > Output stream name**: AggregateDailySummary
    <br>- **Aggregate Settings > Group by**: Select the following columns:
    * **PickUpDate**
    * **PaymentType**
    * **Borough**
    * **Zone**

    <br>- **Aggregate Settings > Aggregates**: Add the following columns and expressions:

    |Column Name |Expression|
    | -----| ----- |
    |TotalTripCount | ```count()``` |
    |TotalPassengerCount |```sum(PassengerCount)```|
    |TotalDistanceTravelled|```sum(DistanceTravelled)```|
    |TotalTipAmount|```sum(TipAmount)```|
    |TotalFareAmount|```sum(FareAmount)```|
    |TotalTripAmount|```sum(TotalAmount)```|
    
    Your list of aggregates and group by should look like this:

    ![](./Media/Lab2-Image103.png)
    ![](./Media/Lab2-Image104.png)

12. Click on the **+** sign next to the **AggregateDailySummary** transformation and type "Select" in the search box. Click the **Select** transformation.

    ![](./Media/Lab2-Image61.png)

13. On the Select properties, enter the following details:
    <br>- **Select Settings > Output stream name**: RenameColumns
    <br>- **Select Settings > Input columns**: Rename the following columns:
    * **Borough** to **PickUpBorough**
    * **Zone** to **PickUpZone**

    Leave all other columns with their default values.

    ![](./Media/Lab2-Image105.png)

14. Click on the **+** sign next to the **RenameColumns** transformation and type "Sink" in the search box. Click the **Sink** destination.

    ![](./Media/Lab2-Image63.png)

15. On the Sink properties, enter the following details:
    <br>- **Sink > Output stream name**: TaxiDataSummary
    <br>- **Sink > Sink dataset**: SynapseDW_NYCTaxiDataSummary
    <br>- **Settings > Table action**: Truncate table
    <br>- **Settings > Enable staging**: Checked

    ![](./Media/Lab2-Image106.png)

    ![](./Media/Lab2-Image66.png)

16. Now you Publish your Data Flow by clicking Publish all. Your full data flow should look like this:

    ![](./Media/Lab2-Image107.png)

## Create and Execute Pipeline
In this section you will create an Azure Synapse Pipeline to copy and transform data in the following sequence:

* Copy NYC Taxi CSV Data files from shared storage account **MDWResources** to your the **nyctaxidata-raw** container in your **syndtlake*suffix*** storage account;

* Copy NYC taxi location data from the Operation SQL Database directly into the NYC.TaxiLocationLookup table in your Azure Synapse Analytics.

* Use a Mapping Dataflow to transform the source data and generate a daily summary of taxi rides. The resulting dataset will be saved in the NYC.TaxiDataSummary table in your Azure Synapse Analytics. This table is then used as a source for the Power BI report.



1.	In the **Azure Synapse workspace studio** portal click the **Integrate** option on the left-hand side panel. On the top click **+** and then click **Pipeline** to create a new pipeline.

2.	On the New Pipeline tab, enter the following details:
    <br>- **General > Name**: Lab 2 - Transform NYC Taxi Data

3.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image108.png)

4.	From the Activities panel, type “Copy Data” in the search box. Drag the Copy Data activity on to the design surface. This copy activity will copy data files from MDWResources to SynapseDataLake.

5.	Select the Copy Data activity and enter the following details:
    <br>- **General > Name**: Copy Taxi Data Files
    <br>- **Source > Source dataset**: MDWResources_NYCTaxiData_Binary
    <br>- **Sink > Sink dataset**: SynapseDataLake_NYCTaxiData_Binary
    <br>- **Sink > Copy Behavior**: Preserve Hierarchy

6.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image109.png)
    ![](./Media/Lab2-Image110.png)

7.	Repeat the process to create another Copy Data Activity, this time to copy taxi location lookup data from MDWResources to your SQL Data Warehouse.

8.	From the Activities panel, type “Copy Data” in the search box. Drag the Copy Data activity on to the design surface.

9.	Select the Copy Data activity and enter the following details:
    <br>- **General > Name**: Copy Taxi Location Lookup
    <br>- **Source > Source dataset**: NYCDataSets_NYCTaxiLocationLookup
    <br>- **Sink > Sink dataset**: SynapseDW_NYCTaxiLocationLookup
    <br>- **Sink > Pre Copy Script**: 
    ```sql
    truncate table NYC.TaxiLocationLookup
    ```
    <br>- **Settings > Enable staging**: Checked
    <br>- **Settings > Staging account linked service**: PolyBaseStorage
    <br>- **Settings > Storage Path**: polybase

10.	Leave remaining fields with default values.

    ![](./Media/Lab2-Image111.png)
    ![](./Media/Lab2-Image112.png)
    ![](./Media/Lab2-Image124.png)

11.	From the Activities panel, type “Data Flow” in the search box. Drag the Data Flow activity onto the design surface. 

12. On the **Adding Data Flow** blade, select **Use existing Data Flow**. In the **Existing Data Flow** drown-down list, select **TransformNYCTaxiData**. Click **OK**.

    ![](./Media/Lab2-Image114.png)

13.	On the Data Flow activity propertie enter the following details:
    <br>- **General > Name**: Transform NYC Taxi Data
    <br>- **Settings > Run on (Azure IR)**: MappingDataFlowsIR/AutoResolveIntegrationRuntime
    <br>- **Settings > Polybase > Staging linked service**: PolyBaseStorage
    <br>- **Settings > Polybase > Staging storage folder**: polybase / [blank]

    ![](./Media/Lab2-Image125.png)

14. Create two **Success *(green)*** precendence constraints between **Copy Taxi Data Files** and **Transform NYC Taxi Data** and between **Copy Taxi Location Lookup** and **Transform NYC Taxi Data**. You can do it by draggind the green square from one activity into the next one.

    ![](./Media/Lab2-Image116.png)

15.	Publish your pipeline changes by clicking the **Publish all** button.

    ![](./Media/Lab2-Image117.png)

16.	To execute the pipeline, click on **Add trigger** menu and then **Trigger Now**.

    ![](./Media/Lab2-Image118.png)

17.	On the **Pipeline Run** blade, click **OK**.

18.	To monitor the execution of your pipeline, click on the **Monitor** menu on the left-hand side panel.

19.	You should be able to see the Status of your pipeline execution on the right-hand side panel.

    ![](./Media/Lab2-Image119.png)

20.	Click in the Pipeline and you should see information about each activity execution in the pipeline. 

    ![](./Media/Lab2-Image120.png)

21. If you click in **Data flow details (glasses icon)** you should be able to see the current state of the activities and monitor them. 

    ![](./Media/Lab2-Image121.png)



## Visualize Data with Power BI
In this section you are going to use Power BI to visualize data from Azure Synapse Analytics. The Power BI report will use an Import connection to query Azure Synapse Analytics and visualise Motor Vehicle Collision data from the table you loaded in the previous exercise.



1.	Download the Power BI report from the link https://aka.ms/ADPLab2 and save it in the Desktop.
2.	Open the file ADPLab2.pbit with Power BI Desktop.
3.	When prompted to enter the value of the **SynapseSQLEndpoint** parameter, type the full server name: **asaworkspace*suffix*.sql.azuresynapse.net**
4.	Click **Load**.

    ![](./Media/Lab2-Image122.png)

5.	When prompted to enter credentials, select **Database** from the left-hand side panel and enter the following details:
    <br>- **User name**: adpadmin
    <br>- **Password**: P@ssw0rd123!
6.	Leave remaining fields with their default values.
7.	Click **Connect**.

    ![](./Media/Lab2-Image123.png)

8.	Once data finish loading interact with the report by changing the PickUpDate slicer and by clicking on the other visualisations.
9.	Save your work and close Power BI Desktop.

    ![](./Media/Lab2-Image40.png)

## Visualize Data with Power BI in Synapse (Optional)

In this section you will publish your report to Power BI Pro account and then link it with Azure Synapse and see it in the Azure Synapse.


1. If you have not done in the previous lab then go to Power BI portal at https://msit.powerbi.com/ and sign in with the account associated to your Power BI Pro account and create a new workspace with name "LabHandsON" by clicking on **Create a workspace**, enter the name in Workspace name and save.

    ![](./Media/Lab1-Image112.png)


2. On the Power BI Desktop click on the **Publish** button.

    ![](./Media/Lab2-Image126.png)

3. Sign in your account associated with the Power BI Pro account, select the workspace "LabHandsON" and then the publish will start. After finishing you will have report available in the Power BI portal.


4. If you have not created the linked service to Power BI in the previous lab then go back to the Azure Synapse Studio click in the **Manage *(toolcase icon)*** option on the left-hand side panel. Under **Linked services** menu item, click **+ New** to create a new linked service connection to Power BI.

5. On the **New Linked Service** blade, type “Power BI” in the search box to find the **Power BI** linked service. Click **Continue**.

    ![](./Media/Lab1-Image114.png)

6. On the New Linked Service (Power BI) blade, enter the following details:
    <br>- **Name**: PowerBIWorkspaceLink
    <br>- **Tenant**: your PBI Pro account Tenant
    <br>- **Workspace name**: LabHandsON

    ![](./Media/Lab1-Image115.png)

7. In the **Azure Synapse Analytics** portal click in the **Develop *(pages icon)*** option on the left-hand side panel. Drill down the Power BI reports and confirm that we have already our report here.

    ![](./Media/Lab2-Image127.png)
