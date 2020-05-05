# SF Crime Statistics with Spark Streaming

In this project we're provided with a real-world dataset, extracted from Kaggle, on San Francisco crime incidents. The goal is to provide a statistical analysis of the data using Apache Spark Structured Streaming.

Kafka
-----------------

The calls for service as they appear in the Kafka topic:

![Kafka Console Output](/images/kafka-console-output.png)

Spark
-----------------

One instance of a progress report:

![Progress Report](/images/progress_report.png)

The Spark UI while the stream is being processed:

![Spark UI](/images/sparkui_jobs.png)

Example output for each batch:

![Batch Output](/images/batch_output.png)

Questions
-----------------

### 1. How did changing values on the SparkSession property parameters affect the throughput and latency of the data?

The `maxOffsetsPerTrigger` parameter determines how many offsets are read from the Kafka topic for each trigger. This affects the amount or rows processed per second.

Lets take a look at a couple of examples. In this first example we use all 16 threads of my local machine by setting `local[*]`. When we set the `maxOffsetsPerTrigger` to `200` only 200 offsets will be loaded per trigger. In this case, Spark will only process approximately 39 rows per second ("processedRowsPerSecond": 39.131285462727455). If we increase maximum amount of offsets per trigger to 10000 the rows processed increases significantly, up to 1560. 

This is due to the fact that Spark can fully utilize all 16 threads when more rows are processed per trigger. It also means it needs to poll Kafka less as the trigger execution takes slightly longer: 5111ms for 200/trigger vs 6409ms for 10000/trigger.

If we reduce the amount of threads from 16 to 1 (`local[1]`) the throughput will decrease as the processing takes longer to execute. 

### What were the 2-3 most efficient SparkSession property key/value pairs? Through testing multiple variations on values, how can you tell these were the most optimal?

I tried variations of the `maxOffsetsPerTrigger` variable. As discussed in the previous question, 10.000 offsets per trigger results in 1560 rows processed per second. Increasing the offsets to 15.000 increased amount of processed rows to 2371 / second. Increasing it to 25.000 increases it to ~3496 / second. I went all the way up to 800000, achieving a processing speed of 45266 rows / second.

I kept the master at `local[*]` to utilize all my 16 threads and achieve maximum parallelism. I also tried tuning the `spark.default.parallelism` parameter to increase the number of partitions in RDDs returned by transformations like `join`, `reduceByKey` and `parallelize`. In local mode this parameters defaults to the number of cores on the local machine. I tried to use two partitions per core, but that decreased performance slightly. Using 16 in total was the optimal variable.

On my local machine, I ended up with:
`maxOffsetsPerTrigger`: `800.000`
`master`: `local[*]`
`spark.default.parallelism`: `16`
to process 45266 rows / second.

