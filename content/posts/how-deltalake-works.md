---
title: "DeltaLake: A solution to blob-storage data warehousing"
date: 2019-08-09
draft: false
tags: ["development", "spark"]
---

## Why Blob Storage Is Risky

To understand why [DeltaLake](https://delta.io) is so useful it is important to understand how Apache Spark executes writes to non-transactional storage like [Amazon S3](https://aws.amazon.com/s3/) or other blob storage. To do this we are going use a contrived example where every day a user extracts their bank account transactions to a CSV file named `yyyy-MM-dd_transactions.csv`:

| file                      | partition | date     | amount    | description    |
|---------------------------|-----------|----------|-----------|----------------|
|2019-08-04_transactions.csv| 0         |2019-08-04|    500.00 | Pay            |
|2019-08-04_transactions.csv| 0         |2019-08-04|    -21.60 | Uber Eats      |
|2019-08-05_transactions.csv| 1         |2019-08-05|    -55.29 | Uber Eats      |
|2019-08-05_transactions.csv| 1         |2019-08-05|    -10.00 | Movie Ticket   |
|2019-08-06_transactions.csv| 2         |2019-08-06|    -12.99 | Netflix        |
|2019-08-06_transactions.csv| 2         |2019-08-06|    -20.00 | ATM Withdrawal |

After [safely applying](https://arc.tripl.ai/transform/#typingtransform) data types to the data the user wants to write that data as a [Parquet](https://arc.tripl.ai/load/#parquetload) file to Amazon S3 (provided by the `hadoop-aws` library) which is a non-transactional file system.

![Parquet Process](/img/2019/parquet.png)

The actual process happens in two steps:

1. The Spark `Driver` process instructs the `Worker` processes (in this case 3 workers) to calculate the dataset and write the partition(s) allocated to them to a `_temporary` file path on S3 and report success/failure to the `Driver`. Filenames include a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) to *likely* avoid collisions.
1. Assuming all `Worker` processes report success then the `Driver` sequentially executes a `mv` (move) for each temporary file part to move it into the final file path. 

This is actually very risky due to the nature of S3 (and other blob storage systems) as:

1. The `mv` move process is executed by the `Driver` process and offers no `transactional` guarantees meaning that if the network connection is broken or the `Driver` process crashes partway through the list of files to move then the target path will contain a non-consistent dataset (only a subset of the data). Contrast this to a database which would not commit the data if the connection was severed before a transaction `COMMIT` has occured.
1. The `mv` (move) process is executed by the `Driver` process sequentially so it is an `O(n)` problem, meaning it will take longer and longer depending on the number of files to move, increasing the risk of the previous `non-consistent` dataset issues.
1. While the `mv` (move) stage is occuring a consumer may read the data and receive a dataset with a non-consistent state. 

## Solutions

There are a few options to solve this problem:

#### Versioned Dataset

Write the full dataset out each run to a new path like: `s3a://blob_store/transactions/${epoch}_transactions.parquet` then on success update an external mechanism like the `Hive Metastore` to store the path to the dataset as `latest` (essentially the pattern described [here](https://arc.tripl.ai/patterns/#database-inconsistency)).

This solution, whilst not terrible, comes with the drawback of:

- having to write a full dataset each time which is both inefficient and potentially costly
- having to run a full-database to essentially hold pointers to the file path (S3 doesn't support `symlinks`)
- how to know which version is most recent and/or valid without an external store (for consuming)

#### DeltaLake

[DeltaLake](https://delta.io) is a custom (open source) solution developed by [Databricks](https://databricks.com/) which elegantly sidesteps the problem of not having `transactional` guarantees in S3 by using versioned `_delta_log/index` [JSON](https://en.wikipedia.org/wiki/JSON) files alongside the `parquet` to describe which changes (`deltas`) are required to move from one `version` of a dataset to the next. This has been included in [Arc](https://arc.tripl.ai/) as the [DeltaLakeExtract](https://arc.tripl.ai/extract/#deltalakeextract) and [DeltaLakeLoad](https://arc.tripl.ai/load/#deltalakeload) plugin stages.

![DeltaLake Process](/img/2019/deltalake.png)

The actual process happens in three steps (very similar to the normal process): 

1. The Spark `Driver` process instructs the `Worker` processes (in this case 3 workers) to first calculate the dataset then write the partition allocated to them to a `_temporary` `parquet` file path on S3 and report success/failure to the `Driver`.
1. Assuming all `Worker` processes report success then the `Driver` sequentially executes a `mv` (move) to each temporary `parquet` file part to move it into the correct file path.
1. Additionally the `Driver` then writes a `JSON` file into the `_delta_log` directory to describe how to transition from one `dataset` version to the next. 

To understand how the `JSON` pointer works here are the results at each `day`:

Running on `2019-08-04` produces the first version `00000000000000000000.json`.

```json
{"commitInfo":{"timestamp":1565327755045,"operation":"WRITE","operationParameters":{"mode":"Append","partitionBy":"[]"},"isBlindAppend":true}}
{"protocol":{"minReaderVersion":1,"minWriterVersion":2}}
{"metaData":{"id":"fb6b1664-1c2c-4b8e-b499-301960d4a1b7","format":{"provider":"parquet","options":{}},"schemaString":"{\"type\":\"struct\",\"fields\":[{\"name\":\"date\",\"type\":\"date\",\"nullable\":true,\"metadata\":{\"internal\":false,\"nullable\":true,\"description\":\"transaction date\"}},{\"name\":\"amount\",\"type\":\"decimal(10,2)\",\"nullable\":true,\"metadata\":{\"internal\":false,\"nullable\":true,\"description\":\"transaction amount\"}},{\"name\":\"description\",\"type\":\"string\",\"nullable\":true,\"metadata\":{\"internal\":false,\"nullable\":true,\"description\":\"transaction description\"}},{\"name\":\"_filename\",\"type\":\"string\",\"nullable\":true,\"metadata\":{\"internal\":true}},{\"name\":\"_index\",\"type\":\"integer\",\"nullable\":true,\"metadata\":{\"internal\":true}},{\"name\":\"_errors\",\"type\":{\"type\":\"array\",\"elementType\":{\"type\":\"struct\",\"fields\":[{\"name\":\"field\",\"type\":\"string\",\"nullable\":true,\"metadata\":{}},{\"name\":\"message\",\"type\":\"string\",\"nullable\":true,\"metadata\":{}}]},\"containsNull\":true},\"nullable\":true,\"metadata\":{\"internal\":true}}]}","partitionColumns":[],"configuration":{},"createdTime":1565327751167}}
{"add":{"path":"part-00000-c5bc4211-f1e5-447b-b6a6-2b06ac214421-c000.snappy.parquet","partitionValues":{},"size":2642,"modificationTime":1565327755000,"dataChange":true}}
```

Running on `2019-08-05` with `saveMode=Append` results in `00000000000000000001.json`.

```json
{"commitInfo":{"timestamp":1565327830447,"operation":"WRITE","operationParameters":{"mode":"Append","partitionBy":"[]"},"readVersion":0,"isBlindAppend":true}}
{"add":{"path":"part-00000-a173a6c3-09f5-443c-a378-b3270b2846da-c000.snappy.parquet","partitionValues":{},"size":2670,"modificationTime":1565327830000,"dataChange":true}}
```

The important bit to focus on is the `add` operation which tells `DeltaLake` to `add` the data contained in `part-00000-a173a6c3-09f5-443c-a378-b3270b2846da-c000.snappy.parquet` to the previous version (`0`) which contains only `part-00000-c5bc4211-f1e5-447b-b6a6-2b06ac214421-c000.snappy.parquet`. On every 10th write `DeltaLake` will automatically capture a state snapshot so the maximum number of `versions` which need to be applied to the last known snapshot are limited.

The beauty of this is that:

- as the `JSON` index is a single write it is `atomic`/`transactional` (S3 supported) and cannot leave users reading an internally inconsistent dataset.
- by default reading a `DeltaLake` path will return the most recent version.
- the underlying data is just `parquet` which has been well tested and offers excellent performance expected from compressed, columnar storage (and if required can be read directly as with any other `parquet`).
- users are able to 'time-travel' back to previous versions without having to store full copies.

The only real limitation is that to get the full benefit the `DeltaLake` extract mechanism must be used but it has already been included in [Arc](https://arc.tripl.ai/) as the [DeltaLakeExtract](https://arc.tripl.ai/extract/#deltalakeextract) stage.