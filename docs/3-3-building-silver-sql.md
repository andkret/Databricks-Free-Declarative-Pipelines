# 3.3 Building the Silver Layer (Cleaning & Modeling)

Afer we've built the Bronze layer, we'll continue with Silver. For silver you have several options, but I will again keep it simple and do some basic cleaning steps while maintaining the stucture of the data as it is. This gives us the flexibility to build our use cases in Gold on top of that.

## Create a Silver Transformation File

Let's get back to our pipeline by selecting 'Jobs & Pipelines'.

![Jobs & Pipelines menu item](/images/3-2/open-jobs-and-pipelines.png)

Then select 'online_retail'.

![Our previously built pipeline](/images/3-3/sql-specific/pipelines-sql.png)

Now click on 'Edit pipeline'.

![Edit pipeline button](/images/3-3/sql-specific/edit-pipeline-sql.png)

In the newly opened browser tab, make sure the workspace is selected.

![Selected workspace tab](/images/3-3/sql-specific/workspace-selected-sql.png)

Now create a new file in your directory '01_silver' by right-clicking on the folder and selecting 'Create file'.

Name the file `clean_orders`, again use SQL as the language but this time, we're going to use the template and not start from scratch. So select `Streaming table` as the 'Dataset type'.

![File creation dialog for Silver cleaning](/images/3-3/sql-specific/create-silver-sql.png)

The template starts with a basic streaming table from a path. We, however, want to use our previously created bronze table. Edit the SQL to look like this:

```sql
/* 
    Cleaned the data by:
*/
CREATE OR REFRESH STREAMING TABLE retail_pipeline.silver.cleaned_orders
AS SELECT *
FROM STREAM(retail_pipeline.bronze.raw_orders);
```

## Rename Columns With Databricks Assistant

Then let's rename our fields to use lower case for all fields and not mix with upper case.

That is a lot of manual coding. Luckily we can use Databricks Assistant to our help. Use this prompt (via `CMD` + `I`):

```
Rename all columns from raw_orders that currently use upper case to lower case using AS aliases. E.g. InvoiceNo AS invoice_no, etc.
```

Then send the prompt. The result may differ for you but I got this very helpful response:

![Renaming result of Databricks Assistant](/images/3-3/sql-specific/renaming-assistant-sql.png)

Since the above code is very helpful, I do accept it via `Enter`. I will also add this in the comment.

Your code should now look like this:

```sql
/* 
    Cleaned the data by:
    - renaming columns
*/
CREATE OR REFRESH STREAMING TABLE retail_pipeline.silver.cleaned_orders
AS SELECT
  InvoiceNo AS invoice_no,
  StockCode AS stock_code,
  Description AS description,
  Quantity AS quantity,
  InvoiceDate AS invoice_date,
  UnitPrice AS unit_price,
  CustomerID AS customer_id,
  Country AS country,
  ingestion_timestamp,
  source_file
FROM STREAM(retail_pipeline.bronze.raw_orders);
```

## Handle Type Conversions With Databricks Assistant

That was only a start with Databricks Assistant and cleaning our data.

In the next step, let's try to do our type conversions with Databricks Assistant. You probably know, context is key when it comes to prompts. So let't use this clear instruction (again use `CMD` + `I`):

```
Update these current columns in the select: customer_id should be STRING, unit_price should be a DECIMAL which makes sense for prices, and invoice_date should be a timestamp in the format M/d/yyyy H:mm
```

This will create more or less what we need:

![Prompt to do the type casting](/images/3-3/sql-specific/type-casting-prompt-sql.png)

However, the result will have duplicated columns which won't work.

> [!NOTE]
> Always validate the result of AI-coding assistants and check if they do what you want. Also the result may differ depending on which line you started the prompt. So best to use a fresh line after 'source_file'.

Nevertheless, accept it, and then replace the existing columns that before did the renaming with our casting steps and add a comment above. So your code should look like this:

```sql
/* 
    Cleaned the data by:
    - renaming columns
    - type casting columns
*/
CREATE OR REFRESH STREAMING TABLE retail_pipeline.silver.cleaned_orders
AS SELECT
  InvoiceNo AS invoice_no,
  StockCode AS stock_code,
  Description AS description,
  Quantity AS quantity,
  TO_TIMESTAMP(InvoiceDate, 'M/d/yyyy H:mm') AS invoice_date,
  CAST(UnitPrice AS DECIMAL(10,2)) AS unit_price,
  CAST(CustomerID AS STRING) AS customer_id,
  Country AS country,
  ingestion_timestamp,
  source_file
FROM STREAM(retail_pipeline.bronze.raw_orders);
```

## Add Derived Fields With Databricks Assistant

Now our Silver data is already in a much better shape. Let us now handle cancellations which are important to really understand the performance of products or the sales.

Do you remember the negative quantities we had in our dataset exploration in chapter 1.2? 

We found a lot of products had negative quantities and that these products' invoice numbers started with a 'C' indicating it is a cancellation.

Let's handle that case by adding a derived field to indicate if something is a cancellation.

Add a new line after 'source_file' and use the following prompt:

```
For all entries with an InvoiceNo field that starts with 'C' or 'c' add a column 'is_cancellation' and set the value to true. For all others, set the value to false.
```

This will generate a very helpful result:

![Cancellation handling](/images/3-3/sql-specific/cancellation-handling-sql.png)

Accept it and after a little formatting and adding a comment on the top, the code should look like this:

```sql
/* 
    Cleaned the data by:
    - renaming columns
    - type casting columns
    - adding a derived column is_cancellation when invoice_no starts with c or C
*/
CREATE OR REFRESH STREAMING TABLE retail_pipeline.silver.cleaned_orders
AS SELECT
  InvoiceNo AS invoice_no,
  StockCode AS stock_code,
  Description AS description,
  Quantity AS quantity,
  TO_TIMESTAMP(InvoiceDate, 'M/d/yyyy H:mm') AS invoice_date,
  CAST(UnitPrice AS DECIMAL(10,2)) AS unit_price,
  CAST(CustomerID AS STRING) AS customer_id,
  Country AS country,
  ingestion_timestamp,
  source_file, 
  CASE WHEN LOWER(invoice_no) LIKE 'c%' THEN true ELSE false END AS is_cancellation
FROM STREAM(retail_pipeline.bronze.raw_orders);
```

Now let's add one more derived column with the total price. Use this prompt:

```
Add a new field 'total_price' that gets calculated by quantity * unit_price?
```

This will generate another field like we defined.

![Prompt for total price](/images/3-3/sql-specific/total_price_prompt-sql.png)

We can accept this as is and after some formatting and adding a comment again on the top, your code should look like this:

```sql
/* 
    Cleaned the data by:
    - renaming columns
    - type casting columns
    - adding a derived column is_cancellation when invoice_no starts with c or C
    - adding a derived column total_price as quantity * unit_price
*/
CREATE OR REFRESH STREAMING TABLE retail_pipeline.silver.cleaned_orders
AS SELECT
  InvoiceNo AS invoice_no,
  StockCode AS stock_code,
  Description AS description,
  Quantity AS quantity,
  TO_TIMESTAMP(InvoiceDate, 'M/d/yyyy H:mm') AS invoice_date,
  CAST(UnitPrice AS DECIMAL(10,2)) AS unit_price,
  CAST(CustomerID AS STRING) AS customer_id,
  Country AS country,
  ingestion_timestamp,
  source_file, 
  CASE WHEN InvoiceNo LIKE 'C%' OR InvoiceNo LIKE 'c%' THEN true ELSE false END AS is_cancellation,
  quantity * unit_price AS total_price
FROM STREAM(retail_pipeline.bronze.raw_orders);
```

This already puts our silver data in a good spot. Now it's time for some final quality checks. That is done with Expactions.

## Expectations

Before we add now these quality checks, we first need to understand what Expectations are in Declarative Pipelines. 

![Databricks Expectations flow graph](/images/3-3/expectations-flow-graph.png)

Expactations are an option to apply quality checks directly in Declarative Pipelines. You define an expectation that you have for your data and the pipeline will check against it.

If the check passes, the record will be kept in all cases. However, you can define how a failure should be handled. You have three options:

1. Warn - This will warn you of a record that doesn't match the expecation but will still keep it.
2. Drop - This will simply drop all values that do not match the expectation.
3. Fail Flow - This will fail the complete flow. Imagine that the whole pipeline should fail when a value is NULL or too high for a single record. 

These Expecations also help with monitoring of the quality of your data and can be applied directly to materialized views and streaming tables like the one we created now in the Silver layer. However, the quality metrics will only be available for warnings or dropped values, not for failed flows.

The Expectation syntax consists of standard SQL Boolean statements for the condition and a name. Lastly, you can even reuse them or group them so if one Expectation fails, the records will be handled accordingly to their failure action (warn, drop, or fail flow).

Syntax:
Python
```python
@dp.expect(<constraint-name>, <constraint-clause>)
```
SQL:
```sql
CONSTRAINT <constraint-name> EXPECT ( <constraint-clause> )
```

## Apply Expectations to Ensure Quality

We will add two Expectations.

1. We will make sure that both invoice ID and stock code are not null.
1. Also we will ensure that the price is greater or equal to zero.
1. We'll want a warning for rows where the customer_id is null

You can add more checks later to make the Silver layer more robust on your own.

The final code should look like this:

```sql
/* 
    Cleaned the data by:
    - renaming columns
    - type casting columns
    - adding a derived column is_cancellation when invoice_no starts with c or C
    - adding a derived column total_price as quantity * unit_price
    - enforce that both invoice ID and stock code are not null
    - enforce that the unit price is greater than or equal to zero
*/
CREATE OR REFRESH STREAMING TABLE retail_pipeline.silver.cleaned_orders(
    CONSTRAINT valid_order EXPECT (invoice_no IS NOT NULL AND stock_code IS NOT NULL) ON VIOLATION DROP ROW,
    CONSTRAINT unit_price_non_negative EXPECT (unit_price >= 0),
    CONSTRAINT customer_id_warn_null EXPECT (customer_id IS NOT NULL)
)
AS SELECT
  InvoiceNo AS invoice_no,
  StockCode AS stock_code,
  Description AS description,
  Quantity AS quantity,
  TO_TIMESTAMP(InvoiceDate, 'M/d/yyyy H:mm') AS invoice_date,
  CAST(UnitPrice AS DECIMAL(10,2)) AS unit_price,
  CAST(CustomerID AS STRING) AS customer_id,
  Country AS country,
  ingestion_timestamp,
  source_file, 
  CASE WHEN InvoiceNo LIKE 'C%' OR InvoiceNo LIKE 'c%' THEN true ELSE false END AS is_cancellation,
  quantity * unit_price AS total_price
FROM STREAM(retail_pipeline.bronze.raw_orders);
```

We will drop all records that do not match our expectations. Let's run the pipeline (`CMD` + `Enter` or 'Run pipeline' button -> button works better).

![Successful build of silver layer](/images/3-3/sql-specific/successful-run-silver-sql.png)

This will give us a clean silver table with quality checks that run before, correct data types, consistent column names, and some additional derived columns for the gold layer.

## Inspect The Result

In the UI you can immediately see in the pipeline graph the DAG and that silver depends on bronze and that the build was successful.

First let's check our data quality. Click on the 'View Data Quality' button in the bottom as a column of the 'cleaned_orders' where a warning is shown.

![Data quality metrics button](/images/3-3/sql-specific/data-quality-button-sql.png)

This will open the expectations and you can directly see that two records were filtered out because of a negative unit price.

![Data quality metrics](/images/3-3/sql-specific/data-quality-metrics-sql.png)

Currently, on this screen, it only shows the failed Expectations. But if you want to see all expectations, you can select the 'All' tab.

![All expectations shown](/images/3-3/sql-specific/all-expectations-sql.png)

This also shows that all records had a stock code and an invoice ID.

Next let's look into the data in the data catalog. Use the 'View in Catalog' button in the 'cleaned_orders' row.

![View in Catalog button](/images/3-3/sql-specific/view-in-catalog-button-sql.png)

This will open the catalog in a new browser tab. 

![cleaned_orders table in catalog](/images/3-3/sql-specific/cleaned-orders-in-catalog.png)

Here you can see all columns that we've defined with the correct snake case and data types.

You can play around here a bit with the different tabs, like History, Lineage, and Sample Data. But this gives us already what we need for the gold layer use cases. That's what we will build next.
