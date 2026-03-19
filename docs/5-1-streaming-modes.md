# 5.1 Streaming Modes in Declarative Pipelines

When implementing a near-real-time or continuous data ingestion with Declarative Pipelines, you need to understand different pipeline modes. 

The pipeline modes are independent of the type and can work with both materialized views and streaming tables.

## Triggered vs. Continuous Pipeline Modes

The triggered pipeline mode processes all data and after all relevant tables are refreshed, it stops updating. Each table is refreshed based on the data available when the update started.

Continuous pipeline mode, however, doen't stop running and process new data as it arrives. 

To avoid unnecessary processing, the pipelines monitor dependent Delta Tables to only perform updates when the contents of those dependent tables have changed.

> [!NOTE]
> However, it is important to understand that both of these modes only work when the pipeline runs continuously. The trigger mode only reduces the update frequency but in both modes the pipeline is permanently running.

This table from the Databricks documentation helps visualize the difference.

| Key questions | Triggered | Continuous |
----------------| ----------| ------------- |
| When does the update stop? | Automatically once complete. | Runs continuously until manually stopped. |
| What data is processed? | Data available when the update starts. | All data as it arrives at configured sources. |
| What data freshness requirements is this best for? | Data updates run every 10 minutes, hourly, or daily. | Data updates are desired between every 10 seconds and  a few minutes. |

_Source: [Databricks Documentation](https://docs.databricks.com/aws/en/ldp/pipeline-mode)_

The main difference is between the way it updates and when. Triggered pipelines will only update when triggered and stop updating when all data was processed. Continuous pipelines run on a always-running cluster but that means more consumption and obviously higher costs.

## How to Set Trigger Interval For Continuous Pipelines

There are several ways of setting the trigger interval for continuous pipelines. You can do that in the definition of your streaming table or materialized view or in the pipeline configuration object in the pipeline configuration.

```python
@dp.table(
  spark_conf={"pipelines.trigger.interval" : "10 seconds"}
)
def <function-name>():
    return (<query>)
```

```sql
SET pipelines.trigger.interval=10 seconds;

CREATE OR REFRESH MATERIALIZED VIEW TABLE_NAME
AS SELECT ...
```

```JSON
{
  "configuration": {
    "pipelines.trigger.interval": "10 seconds"
  }
}
```

However, the setting in Python or SQL inside the definition of the streaming table or materialized view will only set the mode for that single object. The settings configuration will set the mode for the full pipeline.

## Example: Our Retail Pipeline

With our retail data, here's how the modes compare:

**Triggered (every hour):**
- 100 new orders arrive between 9am-10am
- At 10am, pipeline triggers
- Processes all 100 orders → updates Silver (cleaned) → updates Gold (daily sales metric)
- Pipeline stops updating, waits until 11am
- **Dashboard shows:** Sales as of 10am (up to 1 hour old)

**Continuous (real-time):**
- Orders arrive one-by-one throughout the hour
- Pipeline processes each order within seconds
- Silver and Gold update incrementally with each order
- Pipeline runs the whole time
- **Dashboard shows:** Sales from a few seconds ago (near real-time)

It's the same pipeline code and the same transformations but the execution timing differs.

In Chapter 5.3, we'll build a streaming pipeline using Declarative Pipelines. But first, in Chapter 5.2, we need to connect AWS Kinesis to Databricks as our streaming data source. Once connected, we'll run the pipeline in triggered mode, then switch to continuous mode to see near-real-time updates in action.