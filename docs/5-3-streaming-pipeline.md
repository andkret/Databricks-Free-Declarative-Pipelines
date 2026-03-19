# 5.3 Building a Simple Streaming Pipeline with Declarative Pipelines

To finish this course off, let's build a simple streaming pipeline with Lakeflow Declarative Pipelines.

This time, we will consume some sensor temperature data from Kinesis for some near real-time monitoring.

Before we start, let us quickly create a new catalog. Open the catalog explorer option.

![Open the catalog explorer](/images/5-3/open-catalog-explorer.png)

Then select 'Create' and 'Create a catalog'.

![Create a new catalog](/images/5-3/create-catalog-menu.png)

Name it `sensor_pipeline` and keep the type and location as is before selecting again 'Create'.

![Enter the name and hit create](/images/5-3/create-catalog-dialog.png)

Then use 'Create schema' to create also a `bronze` and `silver` schema.

![Create bronze and silver schema](/images/5-3/create-new-schema.png)

That's it for the setup. Next go back to 'Jobs & Pipelines'.

![Open the Jobs & Pipelines](/images/5-3/open-jobs-n-pipelines.png)

Then select 'ETL pipeline'.

![Create another ETL pipeline](/images/5-3/create-etl-pipeline.png)

Now set the pipeline name to `sensor_pipeline` and select as catalog and schema 'sensor-pipeline' and 'bronze'.

![Name the pipeline and set the catalog and schema](/images/5-3/name-the-pipeline.png)

This time we will use Python for the ingestion. Select 'Start with sample code in Python'.

![Start with Python template](/images/5-3/start-with-python-template.png)

This gives us a nice structure. However, we won't need most of it. Remove and create files and folders until you end up with this structure:

![Beginning structure of our Python streaming pipeline](/images/5-3/stream-start-structure.png)

## Read The Kinesis stream

Now open the file 'ingest_sensor_stream.py' and add the following code:

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StructField, StringType, DoubleType

sensor_schema = StructType([
    StructField("timestamp", StringType(), True),
    StructField("sensor_id", StringType(), True),
    StructField("location", StringType(), True),
    StructField("temperature_celsius", DoubleType(), True),
])

@dp.table(
    name="sensor_pipeline.bronze.sensor_logs_raw"
)
def ingest_sensor_logs_raw():
  return (spark.readStream
    .format("kinesis")
    .option("streamName", "sensor-readings")
    .option("region", "eu-central-1")
    .option("serviceCredential", "kinesis-sensor-credential")
    .option("initialPosition", "trim_horizon")
    .load()
    .select(
        F.from_json(F.col("data").cast("string"), schema=sensor_schema).alias("data"),
        F.col("approximateArrivalTimestamp").alias("arrival_timestamp")
    )
    .select(
        F.col("data.timestamp").cast("timestamp").alias("timestamp"),
        F.col("data.sensor_id").alias("sensor_id"),
        F.col("data.location").alias("location"),
        F.col("data.temperature_celsius").alias("temperature_celsius"),
        F.col("arrival_timestamp"),
        F.current_timestamp().alias("ingested_at") 
    )
  )
```

You can see, Lakeflow Declarative Pipelines support both, SQL and Python. A streaming table is just created with the table decorator. We provide an additional name to control how the file will be named and where it should be stored. Even before that we define the schema of the data field.
Then we read the Kinesis stream by using our credentials. You may need to change your region, if it differs from 'eu-central-1'.

Then we only have to get to the data field from the Kinesis message and parse it into a format we can work with. We also use the field of the arrival timestamp. Don't confuse that with the actual timestamp field from the data. 'timestamp' is the timestamp when the sensor logged an event. 'arrival_timestamp' is when we received the data in Kinesis. Lastly, we add an 'ingested_at' which is when the data was ingested in Databricks. This will be important to show the streaming modes.

> [!NOTE]
> I added the ingested_at later to improve the demo. However, in the next few screenshots, you won't see the column. But it should be there for you. I later did a full refresh and got it.

And that's already it to get the data into Databricks. Now run the pipeline to get our data into the streaming table.

![Run the streaming pipeline](/images/5-3/run-streaming-pipeline-bronze.png)

As before, we will receive a success status when the pipeline is finished. You know that already, we can jump to catalog here.

![Open the catalog after bronze transformation happened](/images/5-3/open-catalog-for-streaming.png)

Here you can see all the expected columns. Lastly, you can open the 'Sample Data' to see what we got in here so far.

![Open the sample data](/images/5-3/open-sample-data.png)

If you have to, select the compute. Then you will see one value (in the image two because I ran the command twice before. For you it should be one).

![Sample data of bronze](/images/5-3/sample-data.png)

## Transform The Data in Silver

Now let's quickly jump back to 'Jobs & Pipelines' to add our Silver transformation.

![Open Jobs & Pipelines](/images/5-3/jobs-and-pipelines-from-catalog.png)

Open our 'sensor_pipeline' again.

![Open the sensor_pipeline](/images/5-3/open-sensor-pipeline.png)

Then click on 'Edit pipeline'.

![Edit the streaming pipeline](/images/5-3/edit-streaming-pipeline.png)

Now let's add a new file in the silver folder. But here is the cool part. With Declarative Pipelines we are not limited to choose between Python and SQL for the whole pipeline. Different team members with different skills can use what they prefer or whatever solves the problem at hand the best.

Let's, therefore, create the file `classify_sensor_status.sql`.

![Create a SQL file for Silver transformation](/images/5-3/create-sql-file-for-silver.png)

Now let's change the code to add a classification which then could be used by monitoring tools to send alerts or just in general show the status of the devices.

Add this code:

```sql
CREATE OR REFRESH STREAMING TABLE sensor_pipeline.silver.classify_sensor_status
AS SELECT
    timestamp,
    sensor_id,
    location,
    temperature_celsius,
    arrival_timestamp,
    ingested_at,
    CASE
        WHEN temperature_celsius > 0.0 THEN 'critical'
        WHEN temperature_celsius > -10.0 THEN 'warning'
        ELSE 'normal'
    END AS status
FROM STREAM(sensor_pipeline.bronze.sensor_logs_raw)
```
Then run the pipeline as well. This should lead again in a success. Again open the catalog.

![Open the catalog for the silver table](/images/5-3/open-catalog-for-streaming-silver.png)

Look for the sample data again.

![Open the sample data](/images/5-3/open-sample-data-silver.png)

As you can see, the status is correctly shown for our Freezer.

![Sample data for the silver layer](/images/5-3/sample-data-silver.png)

This gives us data that can be consumed and monitored.

## Use Triggered Streaming Mode

> [!CAUTION]
> Before this section, make sure to point out that when they leave the settings turned on later, this will result in high costs and it's ok if they just watch you build it.
> Also no continuous in the free version!

While all of the steps so far are great, for streaming data it is not very useful to run the pipeline once a day or manually at all. That's why we need the streaming modes that we looked at in chapter 5.1.

As a final step, we want to set how often our streaming tables are refreshed and how they handle incoming messages. As shown before, you can set the streaming mode in SQL, Python, and JSON-configuration files. However, Python and SQL work only for a single table, to update the entire pipeline we need to do the update in the JSON-configuration.

It's important to point out that in both triggered and continuous streaming modes, the pipeline is set to continuous. You only control with the trigger interval how frequently it checks for updates but it runs the whole time.

Open again 'Jobs & Pipelines'.

![Jobs & Pipelines menu item](/images/5-3/jobs-and-pipeline-after-silver.png)

Again open the 'sensor_pipeline'.

![Open the sensor_pipeline](/images/5-3/sensor-pipeline-before-trigger.png)

Then select 'Edit pipeline'.

![Select 'Edit pipeline'](/images/5-3/edit-pipeline-before-trigger.png)

Next, open the pipeline configuration with the cog wheel.

![Open the pipeline settings with the cog wheel](/images/5-3/pipeline-settings-detail-page.png)

I personally think it's the easiest to use the JSON syntax or the YAML. So click on 'JSON'.

![Select the JSON tab](/images/5-3/json-settings-pipeline-detail.png)

Then add these values:

```json
"configuration": {
  "pipelines.trigger.interval": "10 minutes"
},
"continuous": true
```

> [!IMPORTANT]
> `continuous` should already be there but set to false.

![The correct JSON settings](/images/5-3/interval-trigger-settings.png)

This will cause our pipeline to run the whole but check for updates every 10 minutes. After all data was processed, the updating stops and waits until the 10 minutes have passed. This is the less cost intensive option.

To test this in action, run the following in the AWS CLI (make sure you are signed in via `aws login --profile databricks-course`).

```bash
  aws kinesis put-record \
  --stream-name sensor-readings \
  --data "$(echo -n '{"timestamp":"2026-02-12T10:01:00","sensor_id":"sensor_02","location":"Freezer B","temperature_celsius":-15.5}' | base64)" \
  --partition-key sensor_02 
```

Then ~2-3 minutes later, you should see the pipeline is not running. Send another message with the CLI.

```bash
aws kinesis put-record \
  --stream-name sensor-readings \
  --data "$(echo -n '{"timestamp":"2026-02-12T10:02:00","sensor_id":"sensor_03","location":"Freezer C","temperature_celsius":-8.0}' | base64)" \
  --partition-key sensor_03 
```

And another two minutes later, run this.

```bash
aws kinesis put-record \
  --stream-name sensor-readings \
  --data "$(echo -n '{"timestamp":"2026-02-12T10:03:00","sensor_id":"sensor_01","location":"Freezer A","temperature_celsius":-5.0}' | base64)" \
  --partition-key sensor_01 
```

Then wait a few minutes until your pipeline ran once.

> [!NOTE]
> This could theoretically have bad timing. That's why we send 3 messages, so at least 2 will be in the same ingestion run.

After the run, open again the catalog.

![Open catalog of silver table](/images/5-3/open-catalog-after-trigger-interval.png)

You should see - depending on your timing - 2-3 new rows with the same ingestion_at value.

![The result of the interval trigger ingestion](/images/5-3/interval-processed-result.png)

> [!NOTE]
> In my tests the pipeline was way slower than before when turning on continuous. It went from ~1 minute to 14+ minutes. So be patient. Could have also been a Databricks hiccup.

Often this setup is more than enough. But sometimes, we want that our messages are processed immediately. And for that we will look at the continuous streaming mode.

## Use Continuous Streaming Mode

Now let's go back to the pipeline and the pipeline settings. Select 'Jobs & Pipelines'.

![Jobs & Pipelines after trigger interval](/images/5-3/jobs-and-pipeline-after-trigger-interval.png)

Then the 'sensor_pipeline'.

![Select the sensor_pipeline](/images/5-3/select-sensor_pipeline.png)

This time select 'Settings'.

![Select the settings of the pipeline](/images/5-3/select-settings-pipeline.png)

Again select 'JSON' as the display type.

![Select the JSON tab](/images/5-3/json-settings-pipeline.png)

Now remove the setting before:

```json
"configuration": {
  "pipelines.trigger.interval": "10 minutes"
},
```

Keep 'continuous' set to `true`. Then click 'Save'.

![Settings for continuous ingestion](/images/5-3/settings-before-continuous-ingestion.png)

This will cancel the pipeline run. To see the next run, choose the run option. And select the newest run.

![Choose the next run](/images/5-3/choose-continuous-run.png)

Wait until both tables are in 'idle' status.

![Wait for both tables to be idle](/images/5-3/idle-status.png)

Then go back to the AWS CLI and send another event:

```bash
  aws kinesis put-record \
  --stream-name sensor-readings \
  --data "$"(echo -n '{"timestamp":"2026-02-12T10:04:00","sensor_id":"sensor_03","location":"Freezer C","temperature_celsius":2.5}' | base64)" \
  --partition-key sensor_03 c
```

Pretty much immediately, the pipeline should run and process your data.

![Open the catalog](/images/5-3/open-catalog-continuous-ingestion.png)

After the run, check the catalog again for the new entry.

![Result of our near real-time ingestion](/images/5-3/continuous-ingestion-result.png)

In the 'Sample Data', you should see our newest entry. You can also see that there were just a few seconds between the 'arrival_timestamp' in Kinesis and the 'ingested_at' in Databricks. This gives us near real-time processing of messages.

> [!IMPORTANT]
> After this step immediately kill the run by changing the settings of the pipeline to `"continuous": false`. Otherwise, you will get a bad suprise when you receive a terribly high bill.

This concludes our streaming pipeline that was first triggered manually, then run continuously in 10 minute intervals and ultimately processed the data in near real-time. Especially the last case is mostly used for monitoring devices, transactions, etc. where you need to react quickly. However, don't start here. Only use near real-time streaming when it's really necessary since that comes with higher consumption and is more expensive.