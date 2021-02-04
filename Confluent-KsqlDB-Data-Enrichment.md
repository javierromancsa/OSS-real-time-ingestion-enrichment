## KsqlDB a Streaming Database to Enrich Fata on the Fly
The simplicity and mundanity of using a declarative language like SQL to replace 25 to 50 lines of code in Kafka Streams or in other stream processing system in the data world is a very significant advantage. On top of that the fact that you’re doing most if not all the processing at upstream it makes this technology even more appealing to others services and use cases.
### Points when KsqlDB shine:
- Since each transformation does not destroy the original data but creates new data in new topics by default, we can keep composing new data (via JOINs) out of intermediate transformations easily using SQL queries.
- You can pipe transformed data or events anywhere downstream via Kafka Connect or through any custom application consuming result topics instead of using web hooks or constantly polling for changes in a result index.
- You can scale the queries that match newly arriving data by scaling out ksqlDB and Kafka partitions, whereas some components in Elasticsearch have limited horizontal scalability.
- ksqlDB can process different formats of data (i.e., CSV, JSON, and Apache Avro™) within ksqlDB in the same way. The data does not have to be saved as JSON(ELS) or CSV(ADX) first before being able to query it.
Creating custom user-defined functions (UDFs) or user-defined aggregation functions (UDAFs) is extremely easy and effective in integrating logic for transformation or lookups that don’t fall neatly within plain SQL queries. Some have used UDFs and UDAFs to call out to machine learning models.
