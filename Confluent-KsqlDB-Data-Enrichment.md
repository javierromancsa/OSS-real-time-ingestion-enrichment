## KsqlDB a Streaming Database to Enrich Data on the Fly
The simplicity and mundanity of using a declarative language like SQL to replace 25 to 50 lines of code in Kafka Streams or in other stream processing system in the data world is a very significant advantage. On top of that the fact that you’re doing most if not all the processing at upstream it makes this technology even more appealing to others services and use cases.
### Points when KsqlDB shine:
- Since each transformation does not destroy the original data but creates new data in new topics by default, we can keep enriching new data (via JOINs) out of intermediate transformations easily using SQL queries.
- You can pipe transformed data or events anywhere downstream via Kafka Connect or through any custom application consuming result topics instead of using web hooks or constantly polling for changes in a result index.
- Horizontal scalability comes naturally since you can scale out ksqlDB servers and re-partition your input data into a new stream or increase the number of partitions in the input topic.
- ksqlDB can process different formats of data (i.e., CSV, JSON, and Apache Avro™) within ksqlDB in the same way. The data does not have to be saved as JSON(ELS) or CSV(ADX) first before being able to query it.
- Creating custom user-defined functions (UDFs) or user-defined aggregation functions (UDAFs) is extremely easy and effective in integrating logic for transformation or lookups that don’t fall neatly within plain SQL queries. Some have used UDFs and UDAFs to call out to machine learning models.

## We need to create another topic, with a source file call ratings-json.js. This topic contains rating scores for each movie in our database. To seed the ratings topic from a seed file see below:
```
cat ~/demo-scene/streams-movie-demo/data/ratings-json.js | ~/confluent-5.5.0/bin/kafka-avro-console-producer --property schema.registry.url=http://10.5.5.82:8081 --property value.schema.file=kafka-workshop/rating-raw.avsc --topic avro-ratings --bootstrap-server $kafkabrokers
```
And schema registry creates a new subject and version for it:
![pic](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-action-01.png)
In KsqlDB cli:
```
CREATE STREAM ratings WITH (VALUE_FORMAT='AVRO', KAFKA_TOPIC='avro-ratings');
SET 'auto.offset.reset' = 'earliest'; 
select * from ratings where movie id=l emit changes limit 1;
describe extended ratings;
```
![pic](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-action-02.png)
![pic](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-action-03.png)

## Now that we have rating we can create a materialize view doing a join between movies and ratings:
```
CREATE TABLE tbl_movie_ratings AS SELECT m.title, AVG(r.rating) AS avg_ratings, SUM(r.rating) AS sum_rating FROM ratings r LEFT OUTER JOIN tbl_movies m ON m.movie_id = r.movie_id GROUP BY m.title ;
select * from movie_ratings where Title ='Antz';
```
![pic](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-action-05.png)

## With this event streaming enrich table you can query the table with CLI or do REST API calls and is always updated:
Ksqldb cli  query:
```
select title, avg_ratings from tbl_movie_ratings where avg_ratings >= 8.5 emit changes;
```

![pic](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-action-06.PNG)

REST API call:
```
curl -X "POST" "http://10.5.5.83:8088/query" -H "Accept: application/vnd.ksql.v1+json" -H"Content-Type: application/json" -d $'{ "ksql": "select title, avg_ratings from tbl_movie_ratings where avg_ratings >= 8.5 emit changes;", "streamsProperties": {} }'
```
![pic](https://github.com/javierromancsa/images/blob/main/cp-ksqldb-action-07.PNG)

