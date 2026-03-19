# 3.2 Building the Bronze Layer

## Turn on 'Lakeflow Pipelines Editor'

Now let's head back to Databricks and start building a declarative pipeline. 

The first thing we need to make sure is that the 'Lakeflow Pipelines Editor' is turned on.

It's still an experimental feature but is turned on by default. Let's confirm that we have it activated.

Click to your profile.

![Databricks profile option](/images/3-2/databricks_profile.png)

Go to 'Settings'.

![Databricks settings option](/images/3-2/databricks-settings-option.png)

Navigate to the 'Developer' area.

![Databricks settings' developer option](/images/3-2/databricks-developer-menu-item.png)

Here you can change all kinds of things, like the theme of Databricks, code editor styles, and much more. But we are interested in this option pretty much at the bottom in the 'Experimental Features' seciton.

Make sure 'Lakeflow Pipelines Editor' is turned on.

![Databricks settings Lakeflow Editor Pipelines option](/images/3-2/databricks-settings-lakeflow-pipelines-editor.png)

With the Lakeflow Pipelines Editor enabled, we can now build our declarative pipelines visually. Let's create our first pipeline.

## Create The Catalog and Schemas

As you may recall from the Unity Catalog video, we will use Unity Catalog to have our catalog and schemas for this data.

As mentioned, there are different approaches on how to structure them. You could use environment-based schemas like `dev`, `stage`, `prod`. But for this course we will keep it simple by just having one catalog and bronze, silver, gold as its schemas.

Go back to the 'SQL Editor'.

![SQL Editor menu item](/images/1-2/sql-editor-menu-item.png)

Create a new 'SQL Query'.

![SQL Query button](/images/1-2/sql-query-button.png)

This time, let's give this a name to find it later. Right click the tab.

![Right click tab name in SQL Editor](/images/3-2/right-click-name-sql-editor.png)

Select 'Rename'.

![Rename option in SQL Editor](/images/3-2/rename-sql-editor.png)

Name it 'Catalog and Schema Setup'.

![Renamed SQL Query](/images/3-2/renamed-sql-querty.png)

Now in the query area add this code which will create our catalog and our schemas.

```sql
CREATE CATALOG IF NOT EXISTS retail_pipeline;

CREATE SCHEMA IF NOT EXISTS retail_pipeline.bronze;
CREATE SCHEMA IF NOT EXISTS retail_pipeline.silver;
CREATE SCHEMA IF NOT EXISTS retail_pipeline.gold;
```

Then hit `CMD` + `Enter`.

> [!NOTE] 
> Separating data into bronze, silver, and gold schemas also makes access control easier. For example, you might give analysts read-only access to gold, while data engineers have full access to all layers. We won't configure permissions in this course, but the structure supports it.

Wait for the 'OK' result and then verify that it did work by navigating to the 'Catalog' page.

![Catalog menu item](/images/3-2/catalog-tab.png)

Then, click on the 'retail_pipeline' catalog.

![retail_pipeline Catalog](/images/3-2/retail-pipeline-catalog.png)

Here you can clearly see that we have all three schemas. The default and information_schema schemas are created automatically by Unity Catalog for every catalog - you can ignore them."

That's it for the setup of our environment. Now let's build the pipeline.

## Build Our Bronze Declarative Pipeline

### Upload the data...again

In Chapter 1, we uploaded our retail dataset to the default schema of the workspace catalog for exploration. That was a quick way to look at the data and understand its structure.

Now we're building a more realistic pipeline with proper medallion architecture. For that, we'll upload the CSV to a Volume in our retail_pipeline catalog, and our pipeline will ingest it into a Delta table with audit metadata.

That is a proper way so your production data belongs in properly governed catalogs.

First open again the 'Data Ingestion' page. 

![Data ingestion menu item](/images/3-2/data-ingestion-menu-item.png)

But this time use 'Upload files to a volume'.

![Upload files to a volume button](/images/3-2/upload-files-to-volume.png)

Now you need to select what and where you want to upload it. First drag and drop the CSV file from our dataset to the upload area.

![Upload file area](/images/3-2/upload-file-area.png)

Next, select the catalog you want to upload the file to `retail_pipeline`.

![Select the catalog destination for the file](/images/3-2/select-destination-catalog.png)

Next select `bronze` as the schema. Then you select 'Create volume'

![Create Volume button](/images/3-2/create-volume-button.png)

Then do the following:

1. Type in `source_files` as volume name.
1. Keep `Managed volume` as type since the file will be uploaded directly to Databricks.
1. Click on 'Create'.

![Create Volume dialog](/images/3-2/create-volume-dialog.png)

Lastly, hit the 'Upload' button to upload the file. 

![Upload file button](/images/3-2/upload-file-button.png)

This will take some time. But will show a dialog when it is finished.

You can confirm the upload by opening 'Catalog' -> 'retail_pipeline' -> 'bronze' -> 'source_files':

There you should see the 'Online_Retail.csv'.

![Uploaded CSV file](/images/3-2/uploaded-csv.png)

### Open The Pipeline Editor

Now to create our pipeline, we obviously need to open the Pipeline editor by clicking on the 'Jobs & Pipelines' menu item.

![Jobs & Pipelines menu item](/images/3-2/open-jobs-and-pipelines.png)

Now select the 'ETL pipeline' button.

![ETL Pipeline button](/images/3-2/select-etl-pipelne.png)

This will open a new pipeline. Next type in the name `online_retail`.

![Name your pipeline](/images/3-2/name-your-pipeline.png)

Now, provide a catalog and schema. Change the catalog to `retail_pipeline` and the schema to `bronze`. This is the location where the tables and logs will be added to.

![Dropdown to select catalog and schema](/images/3-2/sql-specific/pipeline-catalog-and-schema-sql.png)

Also make sure that the 'Lakeflow Pipelines Editor' is set to `ON`. 

Then use the button to create an empty file since we want to start from scratch this time.

![Start with an empty file button](/images/3-2/sql-specific/create-empty-file-sql.png)

Now select a location and the language you want to use. 

For the langauge, we will use `SQL`. For the folder path, you can keep it as it is or create its own directory for better structure. For this course, I will keep it simple and leave it at the default location. Click 'Select'.

![Filled out folder path and Python as language](/images/3-2/sql-specific/select-folder-path-and-language-sql.png)

This will create a root folder of your pipeline with the name we provided `online_retail` and the pipeline source folder with the name `transformations` and a single empty file `my_transformation.sql`.

Let's change the structue a bit to make it easier to work with. Right-click on the transformations folder. Then select 'Create folder'.

![Create folder button](/images/3-2/sql-specific/create-medallion-folders-sql.png)

Name it `00_bronze`. 

Repeat this process for `01_silver` and `02_gold` respectively. 

![Medallion folders](/images/3-2/sql-specific/medallion-folders-sql.png)

You could come up with different structures like more use-case centered folders but let's keep it simple.

Now remove the file `my_transformation.sql` by right-clicking and selecting 'Move to Trash'. 

![Move to trash button](/images/3-2/sql-specific/move-to-trash-sql.png)

Confirm the deletion and right click on the folder '00_bronze' and select 'Create file'.

Select `SQL` and then name it `ingest_orders`. For the 'Dataset type' you could select out of several options. 

![Different dataset types](/images/3-2/sql-specific/dataset-types-dialog-sql.png)

But this time, we are going to do it from scratch. So keep `None selected` selected. Therefore, hit the 'Create' button.

![Create button to create pipeline source file](/images/3-2/sql-specific/create-ingest-orders-sql.png)

This will open the the file in the pipeline editor and next to it a pipeline paragraph - also called DAG - which will show the lineage later. If you don't see pipeline graph, you can use the button on the right to open it:

![Pipeline graph](/images/3-2/sql-specific/pipeline-graph-sql.png)

The final structure should look like this:

```
online_retail_pipeline
|--transformations
|----00_bronze
|------ingest_orders.sql
|----01_silver
|----02_gold
```

### Ingest The Data Into Bronze Layer

Now let's write the code to actually load the data from the CSV to a bronze delta table.

Databricks recommends using streaming tables for most ingestion use cases which gives us full capabilities like versioning which is very important for governance.

To get full capabilities from Unity Catalog's history and time travel, we will use a streaming table.

```sql
CREATE OR REFRESH STREAMING TABLE retail_pipeline.bronze.raw_orders
AS
SELECT
  InvoiceNo,
  StockCode,
  Description,
  Quantity,
  InvoiceDate,
  UnitPrice,
  CustomerID,
  Country,
  current_timestamp() AS ingestion_timestamp,
  _metadata.file_path AS source_file
FROM STREAM read_files(
  '/Volumes/retail_pipeline/bronze/source_files',
  format => 'csv',
  header => true,
  schema => 'InvoiceNo STRING, StockCode STRING, Description STRING, Quantity LONG, InvoiceDate STRING, UnitPrice DOUBLE, CustomerID LONG, Country STRING'
);
```

First we define our streaming table and its name.

We do provide the path to the directory `source_files` and the format as `CSV`. We also set the header to true and provide a schema so our values are interpreted correctly.

Since we want to accept all values in Bronze and do our quality checks and cleaning later, we'll allow all values to be nullable. Therefore, we have no further constraints.

Run the pipeline by either using `CMD` + `Enter` or use the `Run Pipeline` button on the top.

![Run pipeline button](/images/3-2/sql-specific/run-pipeline-button-sql.png)

The first run usually takes a while. But you can see the progress on the side and bottom of the editor.

![Pipeline progress](/images/3-2/sql-specific/pipeline-progress-sql.png)

After a quick wait, you should see a successful run with information about the created table.

> [!NOTE] 
> In my tests, the button did work much better than `CMD` + `Enter` so I would stick to that.

### Verify That The Run Did Work

> [!NOTE]
> I didn't update the following section to the new SQL pipeline. The steps are the same (only the table has a different name bronze_orders instead of raw_orders). I updated the the code however.

You can see the success in the bottom but also in the history of the last runs and the DAG.

![Pipeline success areas](/images/3-2/pipeline-success.png)

Let's now confirm this by jumping to the Unity Catalog that we've created using this button:

![Jump to catalog button](/images/3-2/jump-to-catalog.png)

This will show us the newly created delta table directly in Uniy Catalog.

![Bronze in catalog](/images/3-2/bronze-in-catalog.png)

That means now if there is an update to the data that needs to be ingested, this pipeline, as a streaming table, will process only the new or changed rows. But it also gives us history (click on the history tab):

![History with three versions](/images/3-2/history.png)

Here you can see for now that the table was created and a setup step took place and lastly, the data was received as a streaming update. This gives us a total of 3 versions.

We also get the correct lineage (click on lineage tab):

![Lineage of our streaming table](/images/3-2/lineage.png)

[THE NEXT STEPS ARE NOT MANDATORY BUT IMO NICE TO SHOW DELTA VERSIONING IN ACTION]

Now before we move on to our silver layer cleaning and modeling, let's look quickly into the time travel to also see this in action.

Open the `SQL Editor`.

![SQL Editor](/images/1-2/sql-editor-menu-item.png)

Create a new query. Then use this code to get one specific entry in it's current state. 

```sql
SELECT * 
FROM retail_pipeline.bronze.raw_orders VERSION AS OF 2
WHERE InvoiceNo = '577696' 
  AND StockCode = '22197';
```

Remember we are already on the third version (starting with a zero index) since there was a total of three versions for the initial load which includes the table creation and setup.

![Initial description](/images/3-2/version-2-initial.png)

You can see the description is right now 'Popcorn Holder'.

Next change the description by running this SQL statement (select the update SQL to only run this statement and not again the select before!).

```sql
UPDATE retail_pipeline.bronze.raw_orders
SET description = 'BEAUTIFUL POPCORN HOLDER'
WHERE InvoiceNo = '577696' 
  AND StockCode = '22197';
```

Then let's query the new version with a version number 3 (again select only this statement).

```sql
SELECT * 
FROM retail_pipeline.bronze.raw_orders VERSION AS OF 3
WHERE InvoiceNo = '577696' 
  AND StockCode = '22197';
```

This will give us correctly the updated description 'BEAUTIFUL POPCORN HOLDER':

![Version 3 with updated description](/images/3-2/version-3-description.png)

Now, let's run again the first query (again remember selecting that statement), to see that this version is still stored and accessible.

![Version 2 description is still available](/images/3-2/version-2-still-there.png)

That was a huge step in your first Declarative Pipeline and all data is ingested and works with Unity Catalog as a Delta Table.