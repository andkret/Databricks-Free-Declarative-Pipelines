# 6.1 Summary & Next Steps

That's it - you've completed the course on Declarative Pipelines and Lakeflow Designer! Let's recap what you've learned. So, you've learned some of the basics about Databricks like creating a Free Edition account to test out Databricks or how to discover data with Genie, Notebooks, or the SQL Editor.

You learned how Delta Lake and Unity Catalog help you build modern data systems - with governed, discoverable data that has versioning and ACID transactions.

You also learned what the medallion architecture design pattern is and how it supports different roles and progressively improves the quality of your data from ingestion to final usage.

You built a complete data pipeline with Declarative Pipelines where we defined what we want and Databricks takes care of the rest. The pipeline for now supports exactly one use case where we used Databricks Assistant to generate transformation logic and aggregations in Silver and Gold layers.

Later we even used Lakeflow Designer which is an AI-powered, low-code pipeline builder that can help to build pipelines in a more visual way. 

And in the last chapter, we even added a streaming source from AWS to get near real-time data.

So where to go from here?

Well, obviously it's a great idea to build more use cases on top of what we have. You can also do a more advanced modeling, further clean our data by adding more Expectations and much more.

But after that you want to do some more advanced topics and best practices. You can add:

- Version control to your pipeline to get a more professional development setup where you can safely change and break things and collaborate with others.
- Add separate environments for development, staging and production.
- Add automated tests like unit tests to make sure your code behaves like expected.
- Implement CI/CD to streamline the release process of your pipeline while ensuring the quality stays high.
- Further dig-into streaming with Declarative pipelines, AutoCDC, etc.

I hope you enjoyed this course! Remember: watching tutorials is only half the journey - the real learning happens when you build. Pick a dataset you care about, apply these patterns, and see what you can create. Happy building!