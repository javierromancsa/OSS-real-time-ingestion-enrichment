## OSS-real-time-ingestion-enrichment
This repository is to enable and facilitate the deployment in Azure of a full stack real-time ingestion and enrichment platform for analytical and operational workloads focus on OSS technology and low code. 

This reference architecture diagram solved the following use cases:

1. Real-time data analytics, dashboards and adhoc query
2. Real-time stream processing/ETL (Ksqldb and Kafka connect)
3. Real-time action/alerts/downstream feed
4. Operational store integration for web/mobile apps

  ![Architecture](https://github.com/javierromancsa/images/blob/main/images01.JPG)
  
## Brief Summary
This solution will be using Apache Kafka as the primary ingestion plataform. Kafka is becoming very popular in the OSS commnunity becuase of the following properties:
- Can handle large volumes of data and is a highly reliable system, fault tolerant and scalable.
- Can handle large real-time data and messages with very low latency.
- It's a distributed event streaming use by "80% of the Forture 100 companies".

For Data Analytics, Adhoc query, data exploration and 
