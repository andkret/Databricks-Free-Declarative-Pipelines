# 3.4 Building the Gold Layer (Business Aggregations)

Now with cleaned data, we can tackle the last layer of the medallion architecture. The gold layer has the answers to the questions the business has about our data.

Whenver you build a production system, you will start here. Not technically, but with the questions or problems you need to solve. Then you start building your pipelines around that.

For this course, we are going to build a single gold aggregation that we can then visualize. The data we want to have is how many sales did we have per day.

## Create a Gold Transformation File

As a starting point, we go back to Databricks and jump again into 'Jobs & Pipelines'.

![Jobs & Pipelines menu item](/images/3-4/gold-jobs-and-pipelines.png)

Again, select our 'online_retail':

![Online retail pipeline in the list](/images/3-4/sql-specific/gold-online-retail-pipeline-sql.png)

Now on the top select 'Edit pipeline' to open the pipeline editor.

![Edit pipeline button in the online retail pipeline graph](/images/3-4/sql-specific/gold-edit-pipeline-sql.png)

This will open a new tab with the editor. Right click on the folder '02_gold'. Then select 'Create file'.

![Create file dialog on Gold folder](/images/3-4/sql-specific/create-gold-transformation-file-sql.png)

Again, we will use SQL. For the name, we are going to use 'daily_sales'. Lastly, for the dataset type, we go with a materialized view this time.

We use materialized views for Gold aggregations because they are optimized for aggregations. Materialized views cache pre-computed results, making dashboard queries extremely fast. They can often use automatic incremental updates, so when new orders arrive, Databricks only recomputes affected dates and not the entire history. It handles late data correctly and in our case I assume there is no version history needed. Business metrics just need current accuracy, not audit trails of every change.

![Gold file input with name, type, and language](/images/3-4/sql-specific/create-file-input-sql.png)

## Build The Aggregation Logic With Databricks Assistant

Let's start with the basic syntax to read from the silver streaming table.

```sql
CREATE OR REFRESH MATERIALIZED VIEW retail_pipeline.gold.sales_by_day
AS SELECT *
FROM retail_pipeline.silver.cleaned_orders;
```

You have several options here. You can now either go step by step or try to use one prompt with a lot of context and fine-tune it later. Since we already went step-by-step in Silver, let's try to use Databricks Assistant this time to get a lot right from the beginning and then do some minor tweaks.

Use this prompt via (`CMD` + `I`):
```
Add to the existing daily_sales materialized view operators to group the data by date (convert invoice_date to date). 

Exclude cancellations (where is_cancellation = true).

Aggregate these metrics:
- total_orders: distinct count of invoice numbers (invoice_no)
- total_revenue: sum of total_price
- unique_customers: distinct count of customers (customer_id)
- total_items_sold: sum of quantity
```

For me, that created a new materialized view and some repetitive code. But we can improve it from here. Accept the result first.

![Databricks Assistant first result](/images/3-4/sql-specific/databricks-assistant-sales-per-day.png)

You should now have this incorrect code:

```sql
CREATE OR REFRESH MATERIALIZED VIEW retail_pipeline.gold.sales_by_day
AS SELECT *
FROM retail_pipeline.silver.cleaned_orders;
CREATE OR REFRESH MATERIALIZED VIEW retail_pipeline.gold.sales_by_day
AS
SELECT
  DATE(invoice_date) AS order_date,
  COUNT(DISTINCT invoice_no) AS total_orders,
  SUM(total_price) AS total_revenue,
  COUNT(DISTINCT customer_id) AS unique_customers,
  SUM(quantity) AS total_items_sold
FROM retail_pipeline.silver.cleaned_orders
WHERE is_cancellation = false
GROUP BY DATE(invoice_date);
```

Let's first remove the obsolete view on top and use the order_date on bottom instead of again using `DATE(invoice_date)`. Lastly, add a order by even though that is not 100% required and could be also added on consumption.

We should end up with this:

```sql
CREATE OR REFRESH MATERIALIZED VIEW retail_pipeline.gold.sales_by_day
AS
SELECT
  DATE(invoice_date) AS order_date,
  COUNT(DISTINCT invoice_no) AS total_orders,
  SUM(total_price) AS total_revenue,
  COUNT(DISTINCT customer_id) AS unique_customers,
  SUM(quantity) AS total_items_sold
FROM retail_pipeline.silver.cleaned_orders
WHERE is_cancellation = false
GROUP BY order_date;
ORDER BY order_date;
```

Now, we can still add a comment to explain our function a bit.

```sql
/*
Sales per day metrics data mart for business reporting.
    
Aggregates completed orders (excludes cancellations) by date to provide:
- Order date 
- Total unique orders per day
- Total revenue generated
- Number of unique customers who ordered
- Total quantity of items sold
- Duplicate rows removed
*/
CREATE OR REFRESH MATERIALIZED VIEW retail_pipeline.gold.sales_by_day
AS
SELECT
  DATE(invoice_date) AS order_date,
  COUNT(DISTINCT invoice_no) AS total_orders,
  SUM(total_price) AS total_revenue,
  COUNT(DISTINCT customer_id) AS unique_customers,
  SUM(quantity) AS total_items_sold
FROM (
  SELECT DISTINCT *
  FROM retail_pipeline.silver.cleaned_orders
  WHERE is_cancellation = false
)
GROUP BY order_date
ORDER BY order_date;
```

Now let's run the pipeline by either clicking on "Run pipeline" or with `CMD` + `Enter` (preferred clicking since results seemed to be better).

This should lead to a successful pipeline run again.

![Successful gold transformation sales per day](/images/3-4/sql-specific/successful-gold-transformation-sql.png)

## Inspect The Result

Let us again investigate the created view in the catalog. I showed you before how to do it but I want to also share a different way with you.

Click on the catalog tab next to the files and folders:

![Catalog tab in file explorer](/images/3-4/sql-specific/catalog-tab-file-explorer-sql.png)

Then expand the retail_pipeline accordingly. You may need to use the refresh button here but then you should see in the gold layer the materialized view.

![Expand the pipeline to see the gold view](/images/3-4/sql-specific/expand-pipeline-sql.png)

Now we already know the materialized view was correctly created. But you can also hover over sales_per_day and use the three dots menu and use 'Open in Catalog Explorer' to again open a new tab of the catalog.

![Dot menu after hovering the gold view](/images/3-4/sql-specific/dot-menu-to-catalog-sql.png)

In this tab, I want to show you quickly one additional cool AI feature that we skipped so far.

You get a table or view description from AI which helps less technical users understand the data and helps with discoverability. You see here how Databricks recommends a description which is quite good, so let's accept it:

![AI Description of Databricks for our sales_per_day view](/images/3-4/sql-specific/ai-description-sql.png)

You can do the same for our Silver and Bronze tables if you want.

## Build a Quick Visualization

To finish this medallion part off and answer the question of sales per day for our hypothetical stakeholders, we will do a quick visualization of the data.

Jump into the 'SQL Editor'.

![SQL Editor menu item](/images/3-4/sql-specific/sql-editor-sql.png)

Select the workspace (e.g. use `CTRL` + `Option` + `E` or press the button).

![Switch to workspace](/images/3-4/sql-specific/open-workspace-sql.png)

Then use the dot menu (Folder actions) to create a new query.

![Folder actions to create a new query](/images/3-4/sql-specific/folder-actions-sql.png)

Rename the query by right-clicking the previous name to `sales_per_day_query`.

Then add this SQL query to our data.

```sql
SELECT * 
FROM retail_pipeline.gold.sales_by_day
ORDER BY order_date ASC;
```

Then hit `CMD` + `Enter` to run the query.

If you get a prompt, simply select 'Start, attach and run'.

![Start, attach and run option](/images/3-4/sql-specific/start-attach-run-sql.png)

Wait until the query finished running with success.

![Successful query](/images/3-4/sql-specific/successful-query-sql.png)

Now as previously, select the '+' icon to add a visualization.

![Add visualization button](/images/3-4/sql-specific/add-visualization-button-sql.png)

Then set the basics for the chart. Select a 'Line' chart as type and then make sure that you turn off "New charts" since it wasn't working at all in my tests (EDIT: In your workspace it did work, so that was maybe fixed, so you can keep it turned on if it does work).
Set the X column as `order_date` and the Y column `total_revenue` and `Sum` as the aggregation.

![Basic chart configuration](/images/3-4/sql-specific/basic-chart-settings-sql.png)

Now select the 'X axis' tab and set the name to `Date`.

![X-Axis configuration](/images/3-4/x-axis.png)

Then select the tab 'Y axis' and set the name to `Total Revenue`.

![Y-Axis configuration](/images/3-4/y-axis.png)

Then hit the 'Save' button to save your visualization.

![Save button to save the visualization](/images/3-4/gold-save-chart.png)

Now you should see a finished visualization tab and can let stakeholders run that query.

![Finished visualization of our data](/images/3-4/sql-specific/gold-finished-chart-sql.png)

Obviously, in production setting, you will probably use a more sophisticated visualization solution, but for this course this nicely sums up our medallion architecture with the answer to the question how our sales behaved by day. You can obviously add more visualizations or even new gold layer use cases - the data is there.