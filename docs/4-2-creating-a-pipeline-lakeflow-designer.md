# 4.3 Updating the Pipeline Inside Lakeflow Designer

So now it's time to finally look into Lakeflow Designer. In order to do that, let's look at a imaginary use case.

## Use Case: Loyalty Program

Our marketing team wants to launch a customer loyalty program. They've seen it work at other retailers. You know, those programs where you get from Base to VIP status based on how much you spend.
They came to us with a simple question: "What spending thresholds should we set for each tier?"
They already know how they want to split customers:

VIP: Top 1% of spenders
Premium: Top 20%
Plus: Top 50%
Base: Top 100% (everyone else)

What they need from us are the actual amounts. Like, "To be VIP, you need to spend at least £X per year."
They sent us a CSV with their tier definitions. Pretty straightforward. Just the tier names and the percentages. Now it's our job to crunch the numbers and give them something actionable.

We'll use Lakeflow Designer to quickly add that use case and help them solve their problem. By the end, marketing gets a simple table they can take straight into their program design meeting.

## Open The Visual Editor

Let's go back to Databricks. First, we need to get back to 'SQL Editor'. If you don't have it open, click on 'SQL Editor'.

![Open the SQL Editor if needed](/images/4-2/sql-editor-from-sql-editor.png)

Let's create a bit of room by now collapsing the side menu. Click on the three lines on the top.

![Close the side menu with navigation drawer menu](/images/4-2/close-side-menu.png)

Next let's create a new query by using the '+' icon.

![New query via '+' icon](/images/4-2/new-query.png)

Let's give it a good name by right-clicking and renaming it. 

![Rename the query via right-click on its name and then selecting rename](/images/4-2/rename-query.png)

Call it `Loyalty Tiers`. 

Now open the 'Visual' editor which is our Lakeflow Designer.

![Open the visual editor via the tab item in the SQL Editor](/images/4-2/open-visual-editor.png)

Let's make even more room, by closing the secondary menu.

![Close secondary menu via 'X' icon](/images/4-2/close-secondary-menu.png)

## Add a New Table For Our Tiers

Next, let's add the loyalty program data we got from the business for the planned classification of our customers. It's attached to the video. 

Drag and drop the file into the canvas. This will open another upload-file dialog.

![Upload file dialog after drag and drop](/images/4-2/upload-file-dialog.png)

We could upload that file to the same source_files volume as before. But I'd like to avoid this from being processed. We could also keep it in bronze, but I will instead create a new schema for reference files like this. Therefore, click on 'Create volume'.

![Create a new volume](/images/4-2/create-new-volume-button.png)

Call the volume `lookup_files`, keep it as managed volume and also use the same catalog as before 'retail_pipeline'. However, for schema, let's use the drop-down and create a new schema.

![Create new volume configuration](/images/4-2/create-new-volume-config.png)

Call the schema `reference`. Keep the rest as it is. Then hit the 'Create' button.

![Create new schema with button](/images/4-2/create-new-schema-reference.png)

Then again hit on 'Create' to create, but you first need to re-enter the volume name `lookup_files`.

![Create new volume with button](/images/4-2/create-new-volume.png)

Then select 'Upload'.

![Upload button in dialog](/images/4-2/upload-button-loyalty-program.png)

As you can see, the loyalty_program_tiers is immediately available as a node in Lakeflow Designer.

![Loyalty program node in Lakeflow Designer canvas](/images/4-2/loyalty-program-node.png)

## Transform The Tiers to Percentiles

Before, we use the full AI power of Lakeflow Designer, let's first explore the operators we have. 

The business delivered us values as percentages of how many customers should have each status. For example, VIP should be only 1%. We would rather work with percentiles. So let's use an operator to achieve this.

Also our data is so far only available as a CSV. We may want to create a table out of it.

So let's start. Make sure the node is selected then click on 'Operator' in the top right.

![Open operators menu](/images/4-2/open-operators.png)

This will open a dropdown with different operators. You can check them out one-by-one, but we are interested in 'Output'.

![Select the Output operator](/images/4-2/select-output-operator.png)

Now select the correct output location and name. As a name select `loyalty_tiers_raw` and as the 'Output location', chose 'retail_pipeline' for the catalog and the schema 'reference'.

![Output configuration](/images/4-2/output-configuration.png)

> [!NOTE]
> During my tests, the dropdown only worked when I searched for the schema.

Now you could select 'Run' to fill in the data for the table. However, in my tests this didn't work.

Therefore, open again the 'Query' tab.

![Switch to query tab again](/images/4-2/switch-to-query.png)

Then hit the run button there.

![Run the created query](/images/4-2/run-query.png)

Then wait for its completion and then move back to the visual editor.

![Completion of materialized view creation](/images/4-2/completion-of-table-creation.png)

That's also the beauty of Lakeflow Designer. You can always switch between SQL and the visual editor.

After that, our 'loyalty_tiers_raw' will be available.

Now let's transform the data to percentiles. This time hover over the output and select the '+' icon that appears on its right.

![Hover then use the + icon to add an operator](/images/4-2/add-operator-to-node.png)

This will also open a selection of operators. This time select 'Transform'.

![Select the transform operator](/images/4-2/select-transform.png)

This will add a new node. Let's rename the transformation to `loyalty_tiers` and select only the first column with the name. Then hit 'Add a custom column'.

![Add a custom column](/images/4-2/add-custom-column.png)

You can now either enter a natural language description or type the expression yourself. In our case the transformation is quite simple, so let's use the expression syntax.

![Select edit expression](/images/4-2/edit-expression-button.png)

Then simply type in:

```sql
100 - top_percent AS percentile
```

![Expression needed for the transformation](/images/4-2/expression-to-transform.png)

This will automatically fill in what you need. Now you only need to hit 'Apply' and it will update accordingly.

![Apply the percentile transformation](/images/4-2/apply-percentile-transformation.png)

For now, you won't see any output of this transformation. So let's open the 'Preview'.

![Open the preview of the newly added node](/images/4-2/percentile-open-preview.png)

This will give us the expected output as percentiles and you can close the preview again.

![Percentiles correctly calculated](/images/4-2/percentile-output.png)

## Add a Second Source

Let's add our business data next. Use the right click on the canvas and select 'Add operator' and then 'Source'.

![Add a second source to the canvas](/images/4-2/add-second-source.png)

This will create a new source node.

![Second source node added](/images/4-2/second-source-node-added.png)

Now select our silver table via 'retail_pipeline' -> 'silver' -> 'cleaned_orders'.

![Select the second source](/images/4-2/select-second-source.png)

This will lead to the correctly added source.

![Cleaned orders correctly added](/images/4-2/add-second-source-final.png)

## Aggregate By Customer

So far, we only looked at the low-code editor of Lakeflow Designer. Let's now turn to AI. With the cleaned_orders selected, use the AI assistant on the bottom.

Type in:

```
Group by customer_id and calculate the sum of total_price as total_spending. Filter out customers where total_spending is negative or zero.
```

Then hit the 'Send' button.

![Total spending prompt](/images/4-2/total-spending-prompt.png)

As you can see this generated two nodes for me. First, an aggregation and second a filtering. You can also see what each step does when selecting a node. Therefore, click 'Accept all'.

![Result of total spending operators](/images/4-2/total-spending-result.png)

There's one more preparation step we need. Let's get the percentile rank of each customer.

Select 'positive_spending_customers' and then type in the following prompt

```
For each customer, calculate their percentile rank based on total_spending. Customers with higher spending should have higher percentile values (0-100).
```

![Prompt to get the percentile rank for each customer](/images/4-2/percentile-rank-prompt.png)

This will create another transformation node. Accept it.

![Result from our transformation with percentile ranks](/images/4-2/percentile-rank-result.png)

## Combine The Two Sources

But now it gets more complicated. We want to combine both sources and create a table that tells us for each loyalty tier what the minimum spending threshold is.

But with Assistant this is fairly easy. Use this prompt:

```
Combine customers_with_percentile_rank and loyalty_tiers. Assign each customer to the tier with the highest percentile they qualify for (where percentile_rank >= percentile). Create an output: tier_name, min_spend_threshold (minimum spending in that tier), customer_count.
```


Then send the prompt.

![Final prompt for joining our data sources](/images/4-2/final-prompt.png)

This will generate three new operators that will ultimately lead to what we wanted to have all tiers with their thresholds.

Accept the Assistant result.

![Accept the output from Assistant for a join](/images/4-2/accept-result-join.png)

Then watch your final output as a preview.

![Final output in the preview](/images/4-2/final-output.png)

This is exactly the information we can hand over to business to answer the question what threshold they should use for each loyalty tier.

However, let me add two things. 

1. The Base shows the minimum actual spending in that tier, not zero. This is intentional - whether the lowest tier requires a minimum spend or is open to everyone is a business decision. Some loyalty programs start with a tier already unlocked, others require a a certain amount. This pipeline gives business the flexibility to define that in the tier configuration.
1. The data reveals that actually just a single customer would reach the VIP status. That is probably not desirable, but our data clearly answers the question and also gives the business all the information so they can adjust the tiers and re-run the pipeline.


> [!NOTE]
> Other things worth mentioning if time: Full SQL code at the end; Menu in the bottom left to zoom, center canvas, etc.;


