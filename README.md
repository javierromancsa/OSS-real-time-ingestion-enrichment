# OSS real-time ingestion enrichment
This repository is to enable and facilitate the deployment in Azure of a full stack real-time ingestion and enrichment platform for analytical and operational workloads focus on OSS technology and low code. 

This reference architecture diagram solved the following use cases:

1. [Real-time data analytics, dashboards and adhoc query](oss-usecase-01.md) 
2. [Real-time stream processing/ETL (Ksqldb and Kafka connect)](oss-usecase-02.md)
3. [Real-time action/alerts/downstream feed](oss-usecase-03.md)
4. [Operational store integration for web/mobile apps](oss-usecase-04.md)

  ![Architecture](https://github.com/javierromancsa/images/blob/main/ADX-UsagePatterns-v1.gif)
  
## Brief Summary
This solution will be using Apache Kafka as the primary ingestion plataform. Kafka is becoming the de-factor standard technology for event streaming becuase of the following properties:
- It's an Opensource distributed event streaming platform use by "80% of the Forture 100 companies".
- It can handle large volumes of data and is a highly reliable system, fault tolerant and scalable.
- It can handle large real-time data and messages with very low latency.

For near real-time Data Analytics, Adhoc query and data exploration we going to use Azure Data Explorer. Azure Data Explorer is a fast, fully managed and highly scalable data analytics database service for near real-time analysis on large(TBs) volumes of heterogeneous data streaming from applications, websites, IoT devices, and more. It natively export Kusto queries that were explored in the Web UI to optimized dashboards.

For real-time stream processing and streaming ETL we are going to use confluent KsqlDB and Kafka Connect for downstream/upstream streaming data. KsqlDB is an event streaming database that lets you use SQL query language to work with both streams of events (asynchronicity) and point-in-time state (synchronicity). 

## Modules
- [Kafka Connect](Kafka-Connect.md) 
- [Confluent Schema Registry](Confluent-Schema-Registry.md)
- [Confluent KsqlDB](Confluent-KsqlDB.md)
- [Confluent KsqlDB in Action](Confluent-KsqlDB-Data-Enrichment.md)
- [Azure Data Explorer Integration/Downstream](adx-kusto-sink-connector.md)
- [Azure Functions and KEDA](azfunctions-actions-alerts-downstream.md)
