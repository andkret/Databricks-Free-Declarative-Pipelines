# 2.2 Unity Catalog Basics

Now that we understand Delta Lake handles reliable storage, let's talk about how to organize and govern all those Delta tables we're about to create.

Imagine your organization has been using Databricks for a year. You now have:

- 15 different workspaces across departments
- 200+ datasets scattered everywhere
- Teams in 5 countries creating tables independently

There are again several things that can and will go wrong.

- Discovery is hard: A data scientist needs customer churn data but has to ask 6 people, search 4 workspaces, and ultimately waste 2 days, and still isn't sure she found the right table.
- Access control is hard to maintain: A contractor leaves the company. The security team needs to revoke access to financial data. So they manually have to look into 15 workspaces, and spend hours finding permissions.
- Compliance as a requirement: An auditor may ask who accessed PII last month. If you have no centralized logs and answering that takes a long time.
- Governance is difficult: A new data privacy regulation requires masking some fields in all reports. Which tables even contain them?

You could track all of that with a static catalog in Excel or Google Sheet but the spreadsheet solution doesn't work or scale. Here are some problems that could occur:

- What if the person maintaining it gets sick?
- What if they forget to update it?
- What if they have more urgent work?

That happens especially in the worst moments in time.

And that's the problem Unity Catalog solves. It provides a centralized metadata layer that governs everything in one place:

You get several features like: 
- Search & discover: It offers powerful search options that makes it easier to find what you're looking for.
- Fine-grained access: Unity Catalog offers a very flexible attribute based access control (ABAC) that makes it possible that employees can only access the data they should have access too which even works on column level.
- Audit: It tracks every data access automatically. 
- Data Lineage: It allows you to exactly tell where the data is coming from and how it flows and alters through your transformations.

Unity Catalog makes your data discoverable, governable, and compliant.

## What is Unity Catalog?

According to the Databricks documentation:

"Unity Catalog is a centralized data catalog that provides access control, auditing, lineage, quality monitoring, and data discovery capabilities across Databricks workspaces." - [Databricks Documentation](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)

![Databricks' Unity Catalog definition](/images/2-2/unity-catalog-definition-databricks.png)

Unity Catalog manages all Databricks assets: tables, views, notebooks, ML models, and files. It's open-source with broad ecosystem support.

![Unity catalog and how it interoperates with different data formats and platforms](/images/2-2/uc-ecosystem.png)
_Source: [Unity Catalog Documentation](https://docs.unitycatalog.io/)_

## How Unity Catalog organizes data

Unity Catalog organizes data hierarchically using a three-tier namespace:
Metastore (top-level container) -> Catalog (organizational boundary) -> Schema (project/use case grouping) -> Objects (tables, views, functions, volumes, etc.).

![Databricks' Unity Catalog definition](/images/2-2/databricks-unity-catalog-hierarchy-model.png)
_Source: [Databricks Documentation](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)_

The metastore is the top-level container for metadata. You have one metastore per region where you operate Databricks workspaces. It registers all your data assets and provides governance across workspaces in that region.

The Catalogs are the first level of organization. Think of them as major organizational boundaries.

Common catalog structures: by environment (dev/prod), by department (sales/engineering), or by data domain (customer_data/product_data).

The workspace catalog is accessible to all users by default. When you need stricter access control, create dedicated catalogs. 

Schemas (also called databases) sit inside catalogs and typically represent a single project or use case.

Objects are the actual data assets. Those could be:

- Tables: Delta tables with your data
- Views: Saved queries
- Volumes: Non-tabular data (images, PDFs, ML model files)
- Functions: User-Defined functions (UDFs) and could be written in Python for example.
- Models: AI models that are packaged with MLFlow
- etc.

## We used Unity Catalog already

Remember when we uploaded our retail dataset in Chapter 1? We created it at `workspace.default.online_retail`.

That three-level path is Unity Catalog:

Catalog: workspace (default catalog for personal work)

Schema: default (default schema)

Table: online_retail

Unity Catalog has been organizing and governing our data from the start.

## Managed vs. External Tables in Unity Catalog

Unity Catalog manages two types of tables:

**Managed Tables:**
- Databricks controls both metadata AND data files
- Data stored in Databricks-managed storage
- 100% governed by Unity Catalog - no external tool can bypass permissions

Which means it's good for: Sensitive data requiring strict governance, prototypes, tutorials. 

**External Tables:**
- Unity Catalog controls metadata, YOU control data files (in S3/ADLS/GCS)
- Data can be accessed outside Unity Catalog
- Requires securing BOTH Unity Catalog AND cloud storage permissions

Which means it's good for: Data lakes shared across platforms, cost optimization (separate compute/storage).

The governance tradeoff:
- Managed tables = simpler governance (one access control system)
- External tables = more flexibility (share with non-Databricks tools) but more complex security

For this course, we're using managed tables to keep governance simple and focus on pipelines.

## Why Unity Catalog matters for our pipelines

In Chapter 3, Unity Catalog will help in several steps.

- It helps us organize our tables for each layer.
- Control access so an analyst may not access raw data or an executive that can only access the dashboards.
- Track lineage so we can visually see where the data is coming from and where it gets altered.
- Audit trail: It will tell us who queried sensitive data which is important in production-level environments.
- Data quality: Enforce and monitor data quality which makes our system more robust.