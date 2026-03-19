# 3.1 Introduction to Declarative Pipelines & Medallion Architecture

With all the basic knowledge about Databricks and its core features equipped, we can now start building our declarative pipelines.

Additionally, I've already mentioned a few times the medallion architecure, bronze, silver, and gold layers and transformations. Therefore, let's first look deeper into the medallion architecture and why it's one of the most common data design patterns out there and recommended by Databricks. 

## Medallion Architecture

### What is Medallion Architecture?

Databricks, who coined the term, defines the medallion architecture as follows:

> "A medallion architecture is a data design pattern used to logically organize data in a lakehouse, with the goal of incrementally and progressively improving the structure and quality of data as it flows through each layer of the architecture (from Bronze ⇒ Silver ⇒ Gold layer tables)." - _Source: [Databricks Documentation](https://www.databricks.com/glossary/medallion-architecture)_ 

There are a few things that stand out in this definition. It is a design pattern to organize data in a lakehouse but you can apply that equally to data warehouses or lakes. But the main information is that it incrementally and progressively improves the structure and quality of the data while it flows through the system layer-by-layer.

![Medallion architecture](/images/3-1/medallion-architecture-databricks.png)

The layers have different names: Bronze -> Silver -> Gold is most common but also Raw -> Refined ->  Curated is often seen.

### The Problem It Solves

The medallion architecture helps you to get from messy data to trustworthy insights. At the beginning, in the bronze layer, you have raw data that includes duplications, invalid values, entries with missing crucial values, etc.

When you do a good job, then after you moved the data through the whole flow, you end up with data that can be easily used by business to drive decisions without outliers or incorrect values leading to wrong conclusions.

So it doesn't just improve the data quality in each layer, but it also moves closer to the business in each of them. In other words, data becomes increasingly more valuable for your business in each step.

In the gold layer, your data should be close to the business and highly relevant.

### Why Three Layers?

But why three layers instead of just cleaning once? Because different users need different things:

- **Data engineers** need raw data (Bronze) for debugging and reprocessing
- **Data scientists** need clean, structured data (Silver) for model training
- **Business analysts** need pre-aggregated metrics (Gold) for dashboards

Each layer serves a specific user group with a specific quality level.

In Unity Catalog, medallion layers can be organized as schemas:
- [CATALOG].bronze
- [CATALOG].silver
- [CATALOG].gold

This makes the architecture explicit in your data catalog. Now let's look at each layer in detail.

**Bronze**:

The bronze layer is the first layer where your data moves into from external source systems. That happens often directly after the ingestion without any transformations. You usually do the bare minimum here but sometimes you do very basic transformations for schema enforcement or parsing. 

- The structure matches the source system structure combined with some metadata about when the ingestion happened, etc.
- This data still contains errors, duplicates, and missing fields.
- It is mostly an untouched form which makes it a good historical archive that isn't accessed frequently (cold storage).
- You usually don't consume data from this layer besides for debugging or reprocessing of the data.

This makes bronze an important layer. Since the data is mostly unchanged after ingestion, debugging can easily detect whether errors happened before or after the pipeline ran. You can also re-run Silver/Gold transformations without re-ingesting from external systems, saving resources on both source and destination.

**Silver**:

The silver layer changes the data from the bronze layer by cleaning and modeling it. The data becomes useable by the first consumers like data scientists or operational analytics teams.

- The structure gets heavily changed and modeled to your needs. Often normalization (3NF), data vaults, and dimensional modeling will be done in this step.
- Invalid data (like duplicates) gets handled and often removed. You cast data types and your data will be in a clean, useable form.
- Theoretically you could stop here, self-service analytics and ad-hoc reporting would be possible.
- The silver layer is the foundation for several gold layer use cases that can be built on top of it.

The silver layer combines data from different sources into an enterprise-view. The data will be used for ML, advanced analytics by different user groups. This is the point at which your data becomes valuable.

**Gold**:

The gold layer is the last layer in the flow and supports specific use cases with aggregated data that can be easily consumed by business-users and analysts.

- The data is usually project or use case specific and answers one question or covers one topic.
- It is also de-normalized, aggregated, and optimized for fast and easy reading with fewer joins.
- You often see star schema or data marts in this layer.
- This usually answers strategic questions and therefore, needs to be refreshed less often.

Business logic is applied at this stage and the data serves the business and less technical consumers. 

### What Are The Benefits?

There are several benefits to using the medallion architecture.

For example:
- The data is replayale because of the raw data in bronze.
- You can incrementally refine the data. Gold supports use cases with easily useable data for the consumers while silver leaves room for later use cases to be added or advanced analytics, machine learning and others.
- You don't have to clean for each gold layer use case. You clean it once and support many use cases with the cleaned data which also reduces repetitive work and code.
- The data is governed and traceable. You'll have a clear data lineage which helps with the governance of the data.

With Delta Lake and Declarative Pipelines, each layer processes only new or changed data. You don't reprocess the entire Bronze table to update Silver, only the delta (changed rows) flows through, making pipelines efficient even with large datasets.

### When Not to Use

One important note is that the medallion architecture is no one-size-fits-all solution that works all the time. For operational analytics, you can't always wait for the gold layer transformations. In real-time operational systems that need low latency like fraud detection this may just take too much time to be helpful. In these cases, you may use the raw data directly or use specialized streaming architectures. 

Also very simple use cases with one source and one consumer might not justify the complexity that the medallion architecture adds. 

Lastly, sometimes you may also already have clean and structured data from a source system, like a well-governed API where the bronze layer becomes obsolete. 

As with any pattern in data engineering, you have to evaluate whether medallion architecture serves your specific needs before adopting it.

## Lakeflow Spark Declarative Pipelines

Now we finally come to the core of the course with Lakeflow Spark Declarative Pipelines, in short SDP, which updated the formerly known Delta Live Tables (DLT).

SDP is Databricks' framework for building and running data pipelines in Python and SQL where you declare what tables you want and their transformations, and Databricks takes care of the rest. That includes:

- Figures out the execution order
- Handles incremental processing where it only processes new or changed data
- Does orchestration of the flows
- Checks the data quality that you provided via expectations
- Supports both batch and streaming
- Gives observability
- Handles retries, parallelism, and performance optimizations

So you will write what you want and Databricks handles the plumbing and low-level details. That's the difference between procedural and declarative pipelines.

In procedural programming, you specify **how** work should be done and in declarative programming, you describe **what** should be achieved instead.

| Procedural                                 | Declarative                          |
| ------------------------------------------ | ------------------------------------ |
| Harder to use                              | Easier to use                        |
| Complex, and requires manual optimizations | Simpler, system handles optimization |
| High flexibility                           | Lower flexibility                    |

SDP hides complexity and messy details so you can concentrate more on the business rather than pipeline maintenance or writing large amount of code.

When building SDP pipelines, you can also use the Databricks Assistant that generates transformation logic, suggests data quality expectations, and optimizes your queries which makes pipeline development even faster.

### Streaming Tables vs. Materialized Views

#### Streaming Tables

When working with Declarative Pipelines, you will mostly use streaming tables and materialized views.

We will use both of them in this course for different cases.

Streaming tables are Delta tables that work best for most ingestion use cases where you want to incrementally process data. So whenever your flow should handle additional data by appending or upserting it instead of reprocessing the full data. 

The input is a streaming source like files, events in a message bus or other streaming data. Our pipeline will then handle each input exactly once, even if there are multiple refreshes and ultimately write the data into the streaming table. These tables can't be changed from outside of our pipeline.

![Steaming table timeline](/images/3-1/st-timeline.png)
_Source: [Databricks Documentation](https://docs.databricks.com/aws/en/ldp/streaming-tables)_

A good example is this diagram from Databricks. In the first iteration t0 we process three rows - Zach, Zara, Zee. All of them will change the name to lower case. In the second iteration t1, we get a new value of Zev which will be appended to our streaming table. The other rows of t0 won't be reprocessed. In the last iteration t2, we will get again the row for Zach which won't be handled since we already did in t0. But Zoe will be processed.

There are some downsides with this when it comes to changes in the logic, like in the example if all entries should be uppercased now, only new data will be processed like that. In that case a full refresh would be required.

#### Materialized Views

Materialized views on the other hand, are like normal views but instead of recomputing on every query, they cache their result for fast access and refreshes the cache on a specified interval. You query data from a materialized view like from a regular table.

Materialized views will always be correct, even if that means you need a complete recompute of your query. But often updates are incremental and Databricks will try to minimize the cost of updating the materialzied view. Databricks only processes the data that is required to keep the view up to date.

Like with streaming tables, materialized views are defined and updated by a single pipeline and can't be changed or updated outside from it. The metadata about the view are stored in Unity Catalog. The cached data is materialized (that is were the name is coming from) in cloud storage.

Materialized views have also their downsides. Since they are views, they don't keep a history and sometimes a full recompute is required and could be expensive. 

Now that we understand what SDP is and why it's powerful, let's build our first pipeline. We'll start with the Bronze layer - ingesting our retail dataset into a Delta table with proper structure and metadata.