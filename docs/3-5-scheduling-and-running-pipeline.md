# 4.2 Scheduling and Running the Pipeline

So far, we support exactly one Gold layer table. In chapter 4, we want to extend it to support one more use case and build that with the help of Lakeflow Designer. 

However, before we do that, it's a good idea to pause for a second and improve what we already have. Our pipeline moves data from ingestion to consumption. Each layer improves the value of the data to our business users.

This works in a way that whenever we drop a new file with data, and then hit the 'Run pipeline' button, it will update our data.

But let's think about it. That means when Bob, a business user, drops a new data file, he needs to ping us that we trigger a pipeline run. That doesn't scale. 

When new data gets added once a day, you need to hit the button daily. Also, when you are sick, there is may be no one who knows the process. The same is true for your vacations. I'm sure you'd rather spend your vacations in a relaxed state than logging into your company's Databricks once a day and trigger a pipeline.

That's why we want to automate the process and this scheduling is what we will do now.

## Add New Data to Unity Catalog

Let's head back to Databricks. Click on 'Catalog'.

![Open the Catalog menu item](/images/3-5/open-catalog.png)

Expand our 'retail_pipeline' -> 'bronze' -> 'Volumes(1)'. Then click on 'source_files':

![Expanded catalog and volumes](/images/3-5/expand-catalog-and-open-volume.png)

Here you can see our previously uploaded CSV.

![Only our previous file is visible](/images/3-5/only-previous-file.png)

In the notes attached to this video, you will find a very simple [CSV](/Online_Retail_update.csv). Let's look at it:

![Our new CSV file](/images/3-5/csv-file.png)

It contains basically just one entry that we can simply find later.

Back in Databricks, choose 'Upload to this volume'.

![Upload to this volume button](/images/3-5/upload-to-this-volume-button.png)

This will open a dialog to upload files, as we previously did. Again drag & drop the new CSV file into the upload area.

![File after adding it via drag & drop](/images/3-5/drag-and-drop-file.png)

Then select 'Upload'.

![Upload button for the updated file](/images/3-5/upload-updated-file-button.png)

You should then rather quickly see the upload summary and the uploaded file.

![Uploaded file summary](/images/3-5/upload-file-summary.png)

Next, let's check if the entry was correctly processed. Go to 'SQL Editor'.

![SQL Editor menu item](/images/3-5/sql-editor-from-catalog.png)

Then create a new SQL Query by selecting the '+' icon and select 'New query'.

![Create a new SQL query](/images/3-5/create-new-sql-query.png)

Then finally, let's query our Bronze layer data with the following SQL:

```sql
SELECT *
FROM retail_pipeline.bronze.raw_orders
WHERE InvoiceNo = '999999';
```

Execute the pipeline via `CMD` + `Enter`. The result is as expected, no entry is available with this invoice number.

![Empty result for our query](/images/3-5/empty-result-sql-editor.png)

Alternatively, you can verify that by going to 'Jobs & Pipelines'.

![Jobs & Pipelines menu item](/images/3-5/jobs-and-pipelines-menu-item.png)

Then select 'Runs'.

![Runs tab](/images/3-5/runs-tab.png)

In the list, you shouldn't see a recent new run.

![List of runs in Jobs & Pipelines](/images/3-5/new-recent-runs.png)

## Schedule Our Next Pipeline Run

Now let's schedule our pipeline runs. Go back to the 'Jobs & pipelines' tab.

![Select the Jobs & pipelines tab](/images/3-5/jobs-and-pipelines-from-runs.png)

Then open the 'online_retail' pipeline.

![Select the retail pipeline](/images/3-5/select-retail-pipeline.png)

In the pipeline editor, we can now schedule our runs easily. Select 'Schedule' on the top right.

![Schedule button to add a job run schedule](/images/3-5/schedule-button.png)

This will allow you to create a job. Give it a name like `online_retail_refresh`. 

![Set the schedule name](/images/3-5/schedule-name.png)

Now you can either go with a simple or advanced setup. There is no right or wrong. Personally, I often like to have a bit more control over the job runs, so let's go with 'Advanced'.

![Advanced tab for our scheduling](/images/3-5/advanced-tab.png)

This allows us to specify the time when the job should run precisely.

![Advanced scheduling options](/images/3-5/advanced-scheduling-options.png)

We can set the time zone, when exactly the pipeline should run and it's frequency. You can even use the cron syntax which is quite common for orchestration tools.

We will go with a simply setting. Schedule the run 5 minutes in advance to your current time. So we have a bit to set everything up.

![Cadence of schedule set 5 minutes in advance](/images/3-5/set-cadence-of-schedule.png)

One more thing worth noting is that you have 'More options' at the bottom. When you expand this, you can even set notifications if you want to be informed on failed runs, when a run has started or on success via email.

![The job notification options](/images/3-5/job-notification-options.png)

That's very convenient. Feel free to set it if you want to.

Lastly, click 'Create'.

![Create button for the schedule](/images/3-5/create-schedule-button.png)

This will also show the set schedule as a view and show a '(1)' in the 'Schedule' button.

![Successful job creation](/images/3-5/successful-job-creation.png)

Now we have to wait quickly for our job to get triggered and the pipeline to run.

You can also navigate back to your 'Jobs & Pipelines'.

![Back to Jobs & Pipelines overview](/images/3-5/back-to-jobs.png)

And in the list you will see your job and maybe it's already running.

![Our newly created job in the list of Jobs & Pipelines](/images/3-5/job-in-list.png)

Click on it. To also see the detail page of the job.

![The job's detail page](/images/3-5/job-detail-page.png)

After a successful run, you will see a success status if you hover over the run or in the bottom.

![Successful job run](/images/3-5/successful-job-run.png)

Now that the run took place, let's look into the results.

## Monitor The Results

Let's navigate back to the 'SQL Editor' in the side menu.

![SQL Editor menu item](/images/3-5/sql-editor-from-job.png)

We can actually use the same query as before.

![Our query to get bronze record](/images/3-5/sql-query-to-get-bronze-record.png)

Hit `CMD` + `Enter` to run the query.

After the query was executed, you should see our newly added value.

![Bronze layer result with our updated entry](/images/3-5/bronze-layer-result.png)

Just to be on the safe side, let's also check if it is in the silver layer. Update the query to this:

```sql
SELECT *
FROM retail_pipeline.silver.cleaned_orders
WHERE invoice_no = '999999';
```

Again use the keyboard shortcut `CMD` + `Enter` to execute the query.

![Silver layer result with our updated entry](/images/3-5/silver-layer-result.png)

Now you can see the data was also processed in the silver layer. That means our new data entries are processed correctly.

## Wrap Up

With scheduling in place, our medallion architecture is now fully automated. New data lands in our volume, the pipeline runs on schedule, and by the time Bob checks his dashboard in the morning, the freshest data is already there. No Slack messages, no manual triggers, no stress during your vacation. This is a fully automated data pipeline.

In the next section, we'll extend our pipeline with an additional Gold layer use case—this time using Lakeflow Designer to help us build it.
