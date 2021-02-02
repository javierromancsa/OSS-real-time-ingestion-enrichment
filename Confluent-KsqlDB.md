## KsqlDB
Confluent KsqlDB is the streaming SQL engine that enables real-time data processing against Apache Kafka®. It provides an easy-to-use, yet powerful interactive SQL interface for stream processing on Kafka, without the need to write code in a programming language such as Java or Python. KsqlDB is scalable, elastic, fault-tolerant, and it supports a wide range of streaming operations, including data filtering, transformations, aggregations, joins, windowing, and serialization. You can use ksqlDB to build event streaming applications from Apache Kafka topics by using only SQL statements and queries. ksqlDB is built on Kafka Streams, so a ksqlDB application communicates with a Kafka cluster like any other Kafka Streams application.

![diagram](https://github.com/javierromancsa/images/blob/main/ksqldb-diagram.JPG)

## Why KsqlDB in Azure?

## Deploying the KsqlDB server with interactive mode and with an internal azure loadbalancer:
In values.yaml changes to the folowing:
```
## External Access
## ref: https://kubernetes.io/docs/concepts/services-networking/service/#type-loadbalancer
external:
enabled: false
type: LoadBalancer
externalTrafficPolicy: Cluster
port: 8088

## Headless mode
## ref: https://docs.confluent.io/current/ksql/docs/installation/server-config/index.html
ksql:
headless: false
```
Also add your respective brokers, schema registry url, replica count and jmx:
```
replicaCount: 3
prometheus:
  jmx:
    enabled: false
kafka:
  bootstrapServers: "wn2-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092,wn1-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092, wn0-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092"
cp-schema-registry:
  url: "http://mytest02-cp-schema-registry:8081"
```
In template/service.yaml you have add the internal loadbalancer annotation:
```
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```
Then we execute the helm:
```helm install ksqldb01 myhelmcharts/cp-ksql-server/```

kubectl get services
![kubectl get services](https://github.com/javierromancsa/images/blob/main/ksqldb01-service.JPG) 
kubectl get pods
![kubectl get pods](https://github.com/javierromancsa/images/blob/main/ksqldb01-pods.JPG)

## Run Ksqldb CLI 
confluent-5.5.0/bin/ksql http://10.5.5.83:8088
![diagram](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-01.png)

## Creating my first stream
```
CREATE STREAM movies_src_json (id BIGINT, title VARCHAR, release_year INT, unknown_1 INT, country VARCHAR, unknown_2 INT, genres VARCHAR,  actors VARCHAR, director VARCHAR, composers VARCHAR, screenwriters VARCHAR, cinematographers VARCHAR, production_companies VARCHAR) WITH (VALUE_FORMAT='JSON', KAFKA_TOPIC='tblmovies01-movies');
```
![diagram](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-02.png)

## creating the stream using the schema registry:
```
CREATE STREAM movies_src WITH (VALUE_FORMAT='AVRO', KAFKA_TOPIC='avro_tblmovies01-movies');
```
## Re-KEY
Next we need to make sure that each record from this stream is identifiable (or partition by) using a field that is unique. Think of this as if it was a primary key for a SQL database, not the same but it will help you choose a field that at least is fit. In order to do this, we need to rekey this stream and take only the fields you want to use:
```
CREATE STREAM movies_brief WITH (PARTITIONS=8) AS SELECT id movie_id, title, release_year FROM movies_src PARTITION BY id;
```
![diagram](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-03.png)

## Checking the stream
```
SELECT * FROM movies_brief where rowkey=63 emit changes;
```
![diagram](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-04.png)

As you can see the stream is always adding records and these records are inmutable. Each record created has his KEY aligned to the value of the movie_id. It is also important to consider how are we pulling from the source, since we are constantly (every 60 secs) using the bulk mode to pull the whole table in kafka connect, the same movie is been duplicated. The Rowtime is what lets us know they are different records.

## Now that you have a stream with each of its records partitioned by the movie_id field, we can finally create our table. 
```
CREATE TABLE tbl_movies (ROWKEY INTEGER KEY) WITH ( VALUE_FORMAT='avro',KAFKA_TOPIC='MOVIES_BRIEF', KEY='movie_id');
```
![diagram](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-05.png)
We still see duplicated records but that because we are pulling constantly and we are updating every 60 seconds. If I pause the connector i will only see one record with that rowkey(remember to reset the offset ```SET 'auto.offset.reset' = 'earliest';```)

## Let's review what we have done with 3 command lines:
![diagram](https://github.com/javierromancsa/images/blob/main/ksqldb-pipelines.JPG)
### You just built in a matter of minutes a fairly complicated ETL pipeline in which data is being transferred from a input topic to a series of pipes that are changing the nature of the data (re-keying in this case) and finally creating a table where data is always up-to-date and with the fields you need in order to use.

## Below is an example of a materialized table. This table let you do a pull query instead of push query(you need "emit changes" at the end).
```
CREATE table movies_count AS SELECT movie_id, count(movie_id) x_times FROM movies_brief group by movie_id;
```
![diagram](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-06.png)
