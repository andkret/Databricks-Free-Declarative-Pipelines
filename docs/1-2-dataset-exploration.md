# 1.2 Getting Started in Databricks and Exploring the Dataset

## Set Up For Databricks Free Edition

Databricks comes with a free edition that you can sign up for to explore what features are available and all of that with your own data.

In order to sign up, use [this link](https://signup.databricks.com/?provider=DB_FREE_TIER&utm_source=youtube&utm_medium=video&utm_campaign=LearnDataEngineering), it will lead you directly to the Free Edition landing page.

Use your preferred method to sign up - via email, Google or Microsoft account.

You will receive a verification code via email that you need to type in to conclude the account creation.

Next, you will be prompted to select an option for what you are going to use Databricks for. For this course, you can select 'For personal use' which will be free forever but is limited to core features, smaller warehouse sizes and compute.

![Choose between free or work edition](/images/1-2/free-or-work-edition.png)

> [!CAUTION]
> Commercial usage of the free edition is prohibited.

Finish the final step of the account creation by naming your account and set your location before you continue.

After a quick setup, you should see the default Databricks UI.

![Account setup done](/images/1-2/databricks-ui.png)

## Upload The Dataset

In order to upload the dataset, you first have to download the dataset from [Kaggle](https://www.kaggle.com/datasets/carrie1/ecommerce-data?resource=download).

This is a simple dataset consisting of a single CSV file which is enough to showcase declarative Databricks pipelines.

Click on 'Download'.

![Download button on the Kaggle page](/images/1-2/download-kaggle.png)

Download the assets as zip. 

[Download dataset directly from this repository](/files/ecommerce-data.zip)

You need an account, so either sign in or register if you have no account yet. As soon as you're signed in, the download will be possible.

<hr>

Back to Databricks. Next, you open the 'Data Ingestion' tab in Databricks under the 'Data Engineering' section.

![The data ingestion tab in the data engineering section](/images/1-2/data-ingestion-tab.png)

There are multiple ways of uploading files to Databricks. You can use connectors to different systems, Fivetran connectors, and more. 

Feel free to play around and use a reasonable upload method for your work, but for this course, we simply select "Create or modify table".

![Create and modify table option in Databricks Data Ingestion](/images/1-2/create-or-modify-table.png)

Next, unzip your downloaded dataset and drag and drop the CSV-file to Databricks.

![Drag and drop the CSV-file to Databricks](/images/1-2/drag-and-drop-dataset.png)

This will initialize the upload which will take some moments.

After the processing by Databricks, you can see a preview of 50 rows with all 8 columns.

You have now the option to change the catalog, use a different schema or change the table name. You can also set if you want to create a new table or overwrite an existing table. In our case it's enough to keep everything as it is and hit 'Create table' at the bottom.

![Create the table on the preview page](/images/1-2/create-table.png)

This will again take a few moments. You should be now on the 'Catalog' tab with you newly created table correctly shown in your workspace and schema (`workspace` - `default`).

![The catalog explorer with your newly created table](/images/1-2/catalog-explorer.png)

> [!NOTE]
> For production use: Instead of using workspace.default, you'd create a dedicated catalog (e.g., retail_analytics) and schema (e.g., bronze, silver, gold) to organize your data properly. We're using workspace.default here to keep the tutorial simple, but we'll explore proper catalog organization in Chapter 2.2 (Unity Catalog Basics).

## Explore The Dataset Via Notebooks

Notebooks are a great place to explore the dataset a bit further and look at what we're dealing with.

### Create a Notebook

In Databricks, click on 'Workspace'.

![Selected Workspace tab](/images/1-2/workspace.png)

Then select 'Create' -> 'Notebook'.

![Create button on Workspace page](/images/1-2/create-button-workspace.png)
![Dialog with create Notebook option.](/images/1-2/create-notebook.png)

Next, you can give the Notebook a meaningful name, to later remember what you did.

Right click on the current name and chose rename and name it 'Dataset exploration'.

![Unclear named notebook](/images/1-2/rename-notebook.png)
![Rename notebook option.](/images/1-2/rename-notebook-dialog.png)

### Exploration

A Notebook consists of several rows that can contain text or code. You can change their order by drag and drop and split logic into pieces or let AI help you with generating code.

Let's first create a text element as a header.

![Use the +Text button to create a markdown row](/images/1-2/create-title-notebook.png)

This text element block supports markdown. For now, simply type in '# Dataset Exploration' which will create a title. 

Then move the text row above the preexisting Python block.

![Use the drag and drop icon to move the text above the code block.](/images/1-2/drag-and-drop-title-notebook.png)

Next, we will use the existing Python block to explore the data with the help of Spark.

It's important to understand that these codeblocks have access to a Spark session which means you don't need to create the session on your own and you can omit the import of it.

Let's create a row/block for each of the code snippets below:

```python
df = spark.table("workspace.default.online_retail")
df.show(5)
```
Run it by using the run button or the shortcut `CMD` + `Enter` or `CTRL` + `Enter` on Windows.

The result will be a truncated text of the first 5 rows of the dataset. We can already see, the different fields like invoice number stock code, etc.

![Spark's show() method applied on the dataset](/images/1-2/show-notebook.png)

```python
df.limit(5).display()
```

As you can see, you don't need to create the df variable and assign the table to it every time. Once created, all succeeding rows/blocks can access the variable. Compared to `show`, `display` gives us a nicely formatted table consisting of interactive HTML. When we look at the column headers, we can see the used datatypes for each column.

So use `show` for a quick check in scripts, and `display` in notebooks.

![Table of fields of the dataset](/images/1-2/table-fields-notebook.png)

You can see that invoice number, stock code, description, country, and the invoice date were interpreted as strings. Invoice number is fine like that since it is possible that the number has leading zeros. When you use an integer, even when it only consists of numbers, the leading zero would be stripped off.

For the invoice date, we can keep it as a string and later transform in our declarative pipelines.

The quantity is an integer (or bigint to be precise) which makes sense. For customer ID we have the problem we talked about before. It was interpreted as a bigint which strips off leading zeros. We would need to fix that in the ingestion since the data could be lost. But I used a REGEX to check that there are no leading zeros in the customer IDs. Therefore, we can later document and transform it to our needs.

That's very important as a data engineer that sometimes you document these assumptions and clarify potential problems upfront.

```python
df.count()
```

 `count` will give you the number of rows that our dataset has. So about half a million.

```python
df.distinct().count()
```

Instead of `count()`, you can also use `distinct().count()` which gives you the number of distinct rows. So it will not count duplicates. The number is different so our dataset indeed containts duplicates. We can handle them in later cleaning steps.

Even though, we have already a good understanding of the dataset, we can use a few more methods to check how the data does look like.

```python
df.printSchema()
```

`printSchema` will give us again the type information in a more condensed format with information about nullable. No suprises here, but that can be quite handy to quickly check the schema.

Just one note, it doesn't show bigint but instead long. That is because `printSchema` shows the Spark SQL types. However, both represent the same 64-bit integer type.

```python
display(df.dTypes)
```

This will give us a nicely formatted table of all datatypes for the different fields.

Lastly, we will look into some stats about the data we are working with.
```python
df.describe().display()
```

This will give us the count, mean, min, max, and the standard deviation of the fields. Obviously, that doesn't make sense for all of those values. But by looking at the description, we can see that the count is lower than the count of invoice numbers. That means, it's not always filled. Same for the customer ID.

We also see that we have negative quantities and very high quantities. That is fine and we will discover the reasons for this later.

![Output of the describe method](/images/1-2/describe-notebook.png)

You can further analyze the data if you want by grouping them, querying different columns and look for duplicates, but for now we got a good overview about the dataset and what we are dealing with. Next, we will see how we can use SQL to query the data instead of Spark and Python.

## Query Data With SQL

### Query With SQL in Notebooks

Sometimes, it's more straightforward to query some data with SQL. Especially, if you are not yet that familiar with Spark. Therefore, I will show you a few ways on how to query the data using SQL instead.

Still in our previous Notebook, create another code block. Let's query just the number of invoices per country using SQL in Spark.

```python
display(
    spark.sql(
        "SELECT Country, COUNT(DISTINCT InvoiceNo) as invoice_count \
        FROM workspace.default.online_retail \
        GROUP BY Country \
        ORDER BY invoice_count DESC"
    )
)
```

Through that we can easily see that United Kingdom has by far the most invoices.

![SQL in Spark](/images/1-2/sql-in-spark.png)

However, there is another way to completely avoid Spark and writing SQL in Notebooks.

Create another code block and add the following content:

```sql
%sql

SELECT 
    SUBSTRING(InvoiceDate, 1, 7) as month,
    SUM(Quantity * UnitPrice) as revenue
FROM workspace.default.online_retail
WHERE Quantity > 0
GROUP BY month
ORDER BY month;
```

This will give us for each month the sum of all invoices which shows us which month performed better or worse to detect trends.

![SQL in Notebook with %sql](/images/1-2/sql-percent.png)

This is nice but not very easy to read. Here Databricks visualization options shine.

Let's explore that a bit. Next to the 'Table' tab in the interactive table, click the `+` icon and select 'Visualization'.

![Add visualization option](/images/1-2/add-visualization-dialog.png)

For this demo, we will keep the bar chart type. We select as X column the month. Add an Y column as 'revenue' and keep 'Sum' as the aggregate method. This should create a chart like this:

![Add visualization screen with options](/images/1-2/add-visualization-screen.png)

Lastly, hit 'Save' and this will add a new tab to the table. You can rename this tab and also add even more visualizations.

![The newly added visualization tab](/images/1-2/visualization-created.png)

This makes it much easier to see the trends and detect outliers. 

### Query with SQL in SQL Editor

The SQL Editor is better for complex queries you want to save and share with your team. Click the 'SQL Editor' in the side menu.

![SQL Editor menu option](/images/1-2/sql-editor-menu-item.png)

Now select `SQL Query` here on top. 

![SQL Query button](/images/1-2/sql-query-button.png)

Now you can write standard SQL to query your data. On the top you can select the workspace and schema that you want to query against.

![Selection of warehouse and schema](/images/1-2/workspace-schema-selection-sql-editor.png)

We can continue with the prefilled values. Use the following query:
```sql
SELECT 
    InvoiceNo,
    StockCode,
    Description,
    Quantity,
    UnitPrice
FROM online_retail
WHERE Quantity < 0
ORDER BY Quantity ASC
LIMIT 20;
```

You will probably have to select the warehouse to run that query. Either hit the 'Connect' button and select the existing warehouse or hit `CMD` + `Enter` and in the dialog select `Start, attach and run`.

![Attach the correct warehouse in the SQL Editor](/images/1-2/attach-warehouse-sql-editor.png)

The result will give us the 20 most negative quantities.

![SQL Editor result](/images/1-2/sql-editor-result.png)

This gives us both Spark and SQL as an option to query data. However, nowadays Databricks offers Genie which allows us to use natural language to learn more about our data. Let's, therefore, look into that next.

## Use Genie to Get Insights

In Notebooks and SQL Editor you can use the Databricks Assistant to create code that will query your data. But Genie is Databricks' GenAI solution that allows to talk with your data. You can use natural language to get insights and we will use that to further explore our dataset. 

Genie has its own place in Databricks. The Genie Space is best for exploratory conversations about your data without writing any code.

Select the 'Genie' menu option.

![Genie menu item](/images/1-2/genie-tab.png)

Next click 'Create' to create a new Genie space.

![Create a new Genie space](/images/1-2/genie_new-space.png)

You need to connect your data next. Select the table 'online_retail' in the upcoming dialog then select 'Create'.

![Connect data to Genie space](/images/1-2/genie-selection-table.png)

Now you have a chat just like with other GenAI solutions.

![Genie chat window](/images/1-2/genie-chat.png)

For example, you can type in:

```
What date is the newest invoice from?
```

After processing, Genie will provide you a solution.

![Genie's response to the question](/images/1-2/genie-space-response.png)

This confirms that our dataset covers December 2010 through December 2011, as we saw in the Kaggle description.

The Genie Space offers a lot of options, like selecting specific properties as input and more. Feel free to test it out on your own before moving on.
