# Operational Store for Downstream Services

## Deploy a Azure Postgresql instance
- https://docs.microsoft.com/en-us/azure/postgresql/howto-manage-vnet-using-portal
- https://docs.microsoft.com/en-us/azure/postgresql/concepts-data-access-and-security-vnet
### In case you want to deploy the azure postgres with a private endpoint
- https://docs.microsoft.com/en-us/azure/postgresql/howto-configure-privatelink-portal
- https://docs.microsoft.com/en-us/azure/postgresql/concepts-data-access-and-security-private-link

## Create a new topic for this sink inside ksqldb(you could have use the original topic TBL_MOVIE_RATINGS)
```
~/confluent-5.5.0/bin/ksql http://10.5.5.83:8088
```
```
ksql> CREATE TABLE tbl_movie_ratings3 WITH (KAFKA_TOPIC='avro_movie_ratings', VALUE_FORMAT='AVRO') AS SELECT m.title, AVG(r.rating) AS avg_ratings, SUM(r.rating) AS sum_rating FROM ratings r LEFT OUTER JOIN tbl_movies m ON m.movie_id = r.movie_id GROUP BY m.title ;
```

## Now we need to deploy a cp-kafka-connector for this jdbc-sink
```
helm install jdbc-sink-01 myhelmcharts/cp-kafka-connect-sink/
sudo nohup kubectl port-forward svc/jdbc-sink-01-cp-kafka-connect 8086:8083 &
```
## Now we need to create a connector for this jdbc-sink using this file as an example:
```
{

	"name": "jdbc_sink_postgres_01",

	"config": {

		"connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",

		"connection.url": "jdbc:postgresql://jrspostgres.postgres.database.azure.com:5432/postgres?sslmode=require",

		"key.converter": "org.apache.kafka.connect.storage.StringConverter",
		"value.converter": "io.confluent.connect.avro.AvroConverter",
		"value.converter.schema.registry.url": "http://mytest02-cp-schema-registry:8081",
		"key.converter.schemas.enable": "true",
		"connection.user": "javier@jrspostgres",
		"auto.create": "true",
		"connection.password": "",
		"auto.evolve": "true",
		"topics": "avro_movie_ratings",
		"insert.mode": "upsert",
		"tasks.max": 1,
		"pk.mode": "record_key",
		"pk.fields": "MESSAGE_KEY",
		"batch.size": 100,
		"db.timezone": "UTC",
		"table.name.format": "kafka_${topic}",
		"errors.log.enable": "true",
		"errors.log.include.messages": "true",
		"transforms": "addSome",
		"transforms.addSome.type": "org.apache.kafka.connect.transforms.InsertField$Value",
		"transforms.addSome.timestamp.field": "RECORD_TS"
	}

}
```
[link to file](https://github.com/javierromancsa/OSS-real-time-ingestion-enrichment/blob/main/simple-avro-jdbc-sink-upsert-movies.json.)

### Why use "key.converter": "org.apache.kafka.connect.storage.StringConverter" ?
- Only the schema of the message value can be retrieved from Schema Registry. Message keys must be compatible with the KAFKA format to be accessible within ksqlDB. ksqlDB ignores schemas that have been registered for message keys.
- https://docs.ksqldb.io/en/latest/concepts/schemas/#schema-inference

## Create the connector
```
curl -X POST http://localhost:8086/connectors -H "Content-Type: application/json" -d @simple-avro-jdbc-sink-insert-movies-01.json
```
## Use the kubectl logs command to validate the connector is working correctly. Seek to see a similar message like in the picture.
![pics](https://github.com/javierromancsa/images/blob/main/postgresql-02.png)

## Now you can login to the postgres instance and check the new table.
![pics](https://github.com/javierromancsa/images/blob/main/postgresql-03.png)
