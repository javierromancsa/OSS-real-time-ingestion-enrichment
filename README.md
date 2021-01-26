## OSS-real-time-ingestion-enrichment
This repository is to enable and facilitate the deployment in Azure of a full stack real-time ingestion and enrichment platform for analytical and operational workloads focus on OSS technology and low code. 

This reference architecture diagram solved the following use cases:

1. Real-time data analytics, dashboards and adhoc query
2. Real-time stream processing/ETL (Ksqldb and Kafka connect)
3. Real-time action/alerts/downstream feed
4. Operational store integration for web/mobile apps

  ![Architecture](https://github.com/javierromancsa/images/blob/main/images01.JPG)
  
## Brief Summary
This solution will be using Apache Kafka as the primary ingestion plataform. Kafka is becoming the standard technology in the OSS commnunity becuase of the following properties:
- It's a distributed event streaming platform use by "80% of the Forture 100 companies".
- It can handle large volumes of data and is a highly reliable system, fault tolerant and scalable.
- It can handle large real-time data and messages with very low latency.

For near real-time Data Analytics, Adhoc query and data exploration we going to use Azure Data Explorer. Azure Data Explorer is a fast, fully managed and highly scalable data analytics database service for near real-time analysis on large(TBs) volumes of heterogeneous data streaming from applications, websites, IoT devices, and more. It natively export Kusto queries that were explored in the Web UI to optimized dashboards.

For real-time data analytics, stream processing and streaming ETL we are going to use confluent KsqlDB and Kafka Connect. 
