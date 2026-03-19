# 2.1 Delta Lake Essentials (ACID, Time Travel, Versioning)

After we've looked into the basic querying in Databricks and got a good overview of our dataset, we will now look into some of the core features of Databricks. Delta Lake and Unity Catalog.

Let's start with Delta Lake and why it was invented.

## The problem that Delta Lake solves

Imagine you're a data engineer at a retail company. You have transaction data flowing into your data lake from 100 stores, 24/7.

There are several things that could and will go wrong:
- A batch job fails halfway through -> partial files corrupt your dataset
- Two processes write simultaneously -> file conflicts, lost data
- Someone accidentally deletes records -> no way to recover them
- Schema changes (new column added) -> queries break, no one knows when it changed
- You need last week's data for compliance -> it's gone, overwritten

The root problem is that data lakes lack transactions, consistency, and history.

This is the issue that Delta Lake solves.
It adds a transaction log which is the single source of truth that tracks every change.

- Failed writes roll back automatically.
- Concurrent writes don't conflict.
- Schema changes are tracked and must be made explicit.
- You can query data at a certain time or version number.
- Partial files never corrupt your data since the transaction log is the single source of truth.

So in summary, Delta Lake turns your data lake into a reliable database.

Let's look into it in more detail.

## What is Delta Lake?

It's an open-source storage layer built on top of Parquet files. It adds a file-based transaction log that enables ACID transactions, scalable metadata handling, and unified batch and stream processing. All in a single platform. 

It enforces schemas to prevent drift and provides versioning with time travel for auditing and compliance.

Initially designed for complex operations on large datasets, it has since evolved to handle workloads of any size, from small prototypes to massive production systems.

In Databricks, every table you create is a Delta table by default which means you're already using Delta Lake features like ACID transactions and time travel, even if you didn't explicitly enable them. Our retail dataset from the previous chapter is already stored as a Delta table.

## How does it work?

Let's understand how Delta Lake actually works under the hood by walking through a simple example.

Imagine you have a customer table with 3 records:

| CustomerID | Name  | Balance |
|------------|-------|---------|
| 001        | Alice | 1000    |
| 002        | Bob   | 500     |
| 003        | Carol | 750     |

What happens when you store this as a Delta table?

Delta Lake creates two things:

Parquet data files: The actual data files that contain customer records stored in compressed Parquet format.
Transaction log: A `_delta_log/` directory with JSON files tracking every change.

Now let's say Alice makes a purchase, and her balance drops to 900. For traditional data lakes, it will overwrite the entire Parquet file with new data.

However, if the write fails halfway it will lead to a corrupted file or lost data. 
Also you can't access yesterday's balance.

Delta Lake has a different approach. 

It will write a NEW Parquet file with Alice's updated record and add a transaction log entry (JSON file): 00001.json

Simplified:
```json
{
  "add": {
    "path": "00001.parquet",
    "size": 524,
    "modificationTime": 1704124800000
  },
  "remove": {
    "path": "00000.parquet"  
  }
}
```

It logs that the new file was added and the old one was removed. 

The old Parquet file still exists on disk, but the transaction log says "don't read this anymore." That's why you can time travel to old files. They are still there.

If a write fails halfway, the partial Parquet file exists but the transaction log has no entry for it. Queries ignore it. This prevents data corruption.

Every Delta table consists of these parts:
1. Parquet data files
These are the actual customer records in compressed columnar format, Multiple files can exist (old versions, new versions, partitioned data), they are stored in your data lake (Databricks-managed storage, or external like S3/ADLS/GCS).

2. Transaction log (`_delta_log/` directory)
JSON files, one per transaction: 00000.json, 00001.json, etc.

Each file records what changed:
- Files added ("add")
- Files removed ("remove")
- Schema at that moment
- Operation type (INSERT, UPDATE, DELETE, MERGE)

This log tells Delta Lake which Parquet files to read.

3. Checkpoints (automatic snapshots)

By default every 10 transactions, Delta Lake creates a checkpoint (Parquet file), so instead of replaying 1000 JSON logs, read the last checkpoint + recent logs. This speeds up reads dramatically.

4. Metadata (stored in the log)

Contains schema definition (column names, types), table properties (partitioning, constraints), and statistics for query optimization.

The transaction log prevents reading stale data. When you delete rows, Delta Lake creates a new Parquet file with the remaining data and marks the old file as "removed" in the log. Queries read the log first to know which files are current, ensuring correct results even though old files still exist on disk (enabling time travel).

## Delta Lake in Databricks: Already Working for You

Remember our retail dataset from Chapter 1? When you uploaded it and created a table, Databricks automatically stored it as a Delta table.

Delta Lake stores all transactions in a _delta_log/ directory as JSON files. In the free edition, we can't browse them directly, but we can see the versions using SQL:

Go back to the workspace page.

![Workspace in side menu](/images/1-2/workspace.png)

Then select our "Dataset exploration" Notebook.

![Dataset exploration Notebook](/images/1-2/dataset-exploration-notebook.png)

Now add a new code block at the bottom and add this code:

```sql
%sql

DESCRIBE HISTORY workspace.default.online_retail
```

![Output of describe history](/images/2-1/describe-history.png)

Each row here corresponds to one JSON file in the transaction log. This is the transaction history Delta Lake is tracking behind the scenes. Delta Lake isn't something you need to 'turn on' in Databricks, it's the default.

> [!NOTE]
> Production consideration: In real-world deployments, Delta Lake tables are often stored in external cloud storage (AWS S3, Azure ADLS, GCP GCS) rather than Databricks managed storage. This separates compute from storage. For this course, we're using Databricks managed storage to keep setup simple.

## What are the key features?

Before we continue, let's quickly look into the key features of Delta Lake.

### ACID Transactions

First, we have ACID transactions.

ACID is an acronym for 4 properties of database transactions:

**Atomicity:** Transactions complete fully or roll back completely. If Judy sends $200 to Tom but the system crashes after deducting from Judy, the entire transaction rolls back - the $200 returns to Judy's account.

**Consistency:** The system enforces rules. If Judy's balance can't go negative and she tries to send $600 (more than her $500 balance), the transaction is rejected.

**Isolation:** Concurrent transactions don't interfere. If two withdrawals happen simultaneously on Judy's account, each sees a consistent snapshot and executes without conflict.

**Durability:** Once a transaction completes, it persists even if the system crashes. Judy's $200 transfer to Tom, once confirmed, is permanent.

Delta Lake implements ACID through its transaction log, making your data lake as reliable as a traditional database.

### Schema enforcement

Delta Lake enforces schemas on write, which means every write operation must match the table's defined schema.

Delta Lake validates every write against the table schema. Writes fail if:
- New columns appear (unless explicitly allowed)
- Data types don't match (e.g., text in a decimal column)
- Required columns are missing

This prevents schema drift - the silent corruption from inconsistent data formats.

In Chapter 3, schema enforcement ensures our Silver table maintains its expected structure. If Bronze data doesn't match, the pipeline fails immediately rather than silently corrupting Silver.

### Time Travel & Versioning

One of Delta Lake's most powerful features is the ability to query your data as it existed at any point in the past.

Delta Lake enables time travel for compliance audits ("What was this customer's balance on Jan 15?"), error recovery (restore accidentally deleted data), and historical analysis (sales before pricing changes).

But how does that work? Remember our earlier example where Alice's balance changed from $1,000 to $900? Delta Lake didn't overwrite the old Parquet file but instead created a new file with the updated record and marked the old file as "removed" in the transaction log.

This is called copy-on-write. Instead of modifying data in place, Delta Lake writes new files and tracks which files are current. The transaction log acts like Git history.

- Each transaction creates a new version
- The log stores what changed (files added/removed), not the full dataset
- You can "replay" the log to reconstruct any past version

Querying past versions is fairly easy.

You can query data by version number:
```sql
SELECT * FROM customers VERSION AS OF 5
```

Or by timestamp:
```sql
SELECT * FROM customers TIMESTAMP AS OF '2025-01-15'
```

That is also great if your data gets corrupted by a bug and you can just go back to a good version and restore it.

However, this comes with a trade-off. Copy-on-write means old Parquet files stick around and consume storage. But storage nowadays gets cheap compared to lost data or failed audits. However, Delta Lake also provides the `VACUUM` command to clean up old versions which can be useful combined with a certain retention period.