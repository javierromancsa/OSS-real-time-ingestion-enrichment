## KsqlDB
Confluent KsqlDB is the streaming SQL engine that enables real-time data processing against Apache Kafka®. It provides an easy-to-use, yet powerful interactive SQL interface for stream processing on Kafka, without the need to write code in a programming language such as Java or Python. KsqlDB is scalable, elastic, fault-tolerant, and it supports a wide range of streaming operations, including data filtering, transformations, aggregations, joins, windowing, and serialization. You can use ksqlDB to build event streaming applications from Apache Kafka topics by using only SQL statements and queries. ksqlDB is built on Kafka Streams, so a ksqlDB application communicates with a Kafka cluster like any other Kafka Streams application.

![diagram](https://github.com/javierromancsa/images/blob/main/ksqldb-diagram.JPG)

## Why KsqlDB in Azure?


## Run Ksqldb CLI 
confluent-5.5.0/bin/ksql http://10.5.5.83:8088
![diagram](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-01.pgn)

## Creating my first stream
```
CREATE STREAM movies_src_json (id BIGINT, title VARCHAR, release_year INT, unknown_1 INT, country VARCHAR, unknown_2 INT, genres VARCHAR,  actors VARCHAR, director VARCHAR, composers VARCHAR, screenwriters VARCHAR, cinematographers VARCHAR, production_companies VARCHAR) WITH (VALUE_FORMAT='JSON', KAFKA_TOPIC='tblmovies01-movies');
```
![diagram](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-02.png)

## creating the stream using the schema registry:
```
CREATE STREAM movies_src WITH (VALUE_FORMAT='AVRO', KAFKA_TOPIC='avro_tblmovies01-movies');
```
## Next we need to make sure that each record from this stream is identifiable (or partition by) using a field that is unique. Think of this as if it was a primary key for a SQL database, not the same but it will help you choose a field that at least is fit. In order to do this, we need to rekey this stream and take only the fields you want to use:
