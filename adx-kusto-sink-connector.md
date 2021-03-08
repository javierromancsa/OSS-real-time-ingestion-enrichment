# Azure Data Explorer
A fast and highly scalable data exploration service for log and telemetry data. A fully managed data analytics service for near real-time analysis on large volumes(TBs) of data streaming from applications, websites, IoT devices, and more. You can quickly identify patterns, anomalies, and trends in your data. Explore new questions and get answers in minutes. Run as many queries as you need, thanks to the optimized cost structure.

## What makes Azure Data Explorer unique?
- Scales quickly to terabytes of data, in minutes, allowing rapid iterations of data exploration to discover relevant insights.
- Offers an innovative query language, optimized for high-performance data analytics.
- Supports analysis of high volumes of heterogeneous data (structured and unstructured).
- Provides the ability to build and deploy exactly what you need by combining with other services to supply an encompassing, powerful, and interactive data analytics solution.

## Deploying Azure Explorer cluster and Creating DB with target tables
Using the following repository you can deploy the cluster and a dababase in your own Vnet : https://github.com/javierromancsa/oss-demo-azure-services
For example below is posible way to deploy this using azure CLI
```
az deployment group create \
  --name adxDeployment \
  --resource-group oss-demo \
  --template-uri https://raw.githubusercontent.com/javierromancsa/oss-demo-azure-services/main/adx-azuredeploy.json \
  --parameters virtualNetworkName='aks-23g4wsrucx4q4Vnet' subnetName='adx-subnet' skuName='Standard_D11_v2' databases_kustodb_name='movies' subnetPrefix='10.1.3.0/24'
```
**Note** If you need futher details about this visit this [link](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-cli?toc=%2Fcli%2Fazure%2Ftoc.json&bc=%2Fcli%2Fazure%2Fbreadcrumb%2Ftoc.json)
Now let's go to our Azure Data Explorer and create the tables:
```
// Create tables
.create table ['movies_ratings_kafka_hdi']  ( ['TITLE']:string, ['AVG_RATINGS']:decimal , ['SUM_RATING']:decimal  )

.create table ['movies_ratings']  ( ['TITLE']:string, ['AVG_RATINGS']:decimal , ['SUM_RATING']:decimal  )

// Create mapping
.create table ['movies_ratings_kafka_hdi'] ingestion json mapping 'movies_ratings_kafka_hdi_mapping' '[{"column":"TITLE","path":"$.TITLE","datatype":"string"},{"column":"AVG_RATINGS","path":"$.AVG_RATINGS","datatype":"decimal"},{"column":"SUM_RATING","path":"$.SUM_RATING","datatype":"decimal"} ]'

.create table ['movies_ratings'] ingestion avro mapping 'movies_ratings_mapping' '[{"column":"TITLE","path":"$.TITLE","datatype":"string"},{"column":"AVG_RATINGS","path":"$.AVG_RATINGS","datatype":"decimal"},{"column":"SUM_RATING","path":"$.SUM_RATING","datatype":"decimal"} ]'

// Batching policy override of defaults, to consume faster
.alter table movies_ratings policy ingestionbatching @'{"MaximumBatchingTimeSpan":"00:00:05", "MaximumNumberOfItems": 20, "MaximumRawDataSizeMB": 300}'

.alter table movies_ratings_kafka_hdi policy ingestionbatching @'{"MaximumBatchingTimeSpan":"00:00:05", "MaximumNumberOfItems": 20, "MaximumRawDataSizeMB": 300}'
```
The movies_ratings_kafka_hdi will be for a json topic without Confluent Schema Registry and the movies_ratings will be for a avro topic with with Confluent Schema Registry

**Note** If you can't connect to the database UI please add your computers public IP to the Azure explorer NSG as inbound rule for https. 

### Now you have to create a App registration in AAD and create a Secret for it. This Service Principal we just created will need the right access to these to tables for ingetisting.
Using this [link](https://docs.microsoft.com/en-us/azure/data-explorer/provision-azure-ad-app) to obtain/copy the appId and the secret key string
```
.add database movies ingestors ('aadapp="yourappId"') 'Kafka Azure AD App'
```
## Deploying the Kafka Connector:

### Download the azure data explorer kafka connector
```
mkdir -p myimages/adx-kafka-connect
cd myimages/adx-kafka-connect/
wget https://github.com/Azure/kafka-sink-azure-kusto/releases/download/v2.0.0/kafka-sink-azure-kusto-2.0.0-jar-with-dependencies.jar
```
### Create a dockerfile for a custom image:
custom-build-image.dockerfile file below:
```
FROM confluentinc/cp-kafka-connect:5.5.0
COPY kafka-sink-azure-kusto-2.0.0-jar-with-dependencies.jar /usr/share/java
```
### Create an image/container
**Note**: You need to attach the azure container registry to this aks cluster using this [link](https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration#configure-acr-integration-for-existing-aks-clusters)
```
cd ~/myimages/adx-kafka-connect/
az login
az acr build --registry jrsacr --image myadx-kafka-connect:v1 -f custom-build-image.dockerfile .
```
### Create the Helm for ADX kafka connector:
cp -R ~/cp-helm-charts/charts/cp-kafka-connect/ myhelmcharts/adx-cp-kafka-connect/
![pic](https://github.com/javierromancsa/images/blob/main/adx-kusto-sink-01.png)

### Edit the replica count based on the topic partition in the file values.yaml:
```
replicaCount: 8
```
### Edit the image that is going to be pull:
```
image: jrsacr.azurecr.io/myadx-kafka-connect
imageTag: v1
```
### Edit the kafka connect properties on this section for the workers:
```
configurationOverrides:
"plugin.path": "/usr/share/java,/usr/share/confluent-hub-components"
"key.converter": "org.apache.kafka.connect.json.JsonConverter"
"value.converter": "org.apache.kafka.connect.json.JsonConverter"
"key.converter.schemas.enable": "false"
"value.converter.schemas.enable": "false"
"internal.key.converter": "org.apache.kafka.connect.json.JsonConverter"
"internal.value.converter": "org.apache.kafka.connect.json.JsonConverter"
"config.storage.replication.factor": "3"
"offset.storage.replication.factor": "3"
"status.storage.replication.factor": "3"
"connector.client.config.override.policy" : "All'
```
### Note: For simplicity I will use the JSON converter but the connector support others. 
Check the link https://github.com/Azure/kafka-sink-azure-kusto#37--kafka-connect-converters

### For further adjusting the values of the values.YAML on the helm config use this link
https://github.com/confluentinc/cp-helm-charts/tree/master/charts/cp-kafka-connect#kafka-connect-deployment

### For now don't enable prometheus as below:
```
prometheus:
  ## JMX Exporter Configuration
  ## ref: https://github.com/prometheus/jmx_exporter
  jmx:
    enabled: false
```
### Get kafka brokers hostnames and add them to the kafka section in value.yml as below:
kafka.bootstrapServers = "wn0-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092,…"

### Optional way to the get this:
```
export clusterName=
export password=
export KAFKABROKERS=$(curl -sS -u admin:$password -G https://$clusterName.azurehdinsight.net/api/v1/clusters/$clusterName/services/KAFKA/components/KAFKA_BROKER | jq -r '["\(.host_components[].HostRoles.host_name):9092"] | join(",")' | cut -d',' -f1,2);
```
### Since the connector support schema registry you could add this but if you don't use it you can leave it blank:
```
cp-schema-registry:
  url: "http://mytest02-cp-schema-registry:8081"
```
### Helm deploy kafka connector
helm install kusto-sink-01 myhelmcharts/adx-cp-kafka-connect/
![pic](https://github.com/javierromancsa/images/blob/main/adx-kusto-sink-02.png)

### start forwarding and check the plugin is available:
sudo nohup kubectl port-forward svc/kusto-sink-01-cp-kafka-connect 8084:8083 &

![pic](https://github.com/javierromancsa/images/blob/main/adx-kusto-sink-03.png)


### Create the kusto sink connector:
### This is an example file:
```
{
        "name": "KustoSinkConnector_01",
        "config": {
                "connector.class": "com.microsoft.azure.kusto.kafka.connect.sink.KustoSinkConnector",
                "kusto.sink.flush_interval_ms": "10000",
                "errors.log.enable" : "true",
                "kusto.tables.topics_mapping": "[{'topic': 'json_movie_ratings','db': 'movies', 'table':'movies_ratings_kafka_hdi','format': 'json', 'mapping':'movies_ratings_kafka_hdi_mapping'}]",
                "aad.auth.authority": "",
                "kusto.ingestion.url":"https://private-ingest-jrsadx.eastus2.kusto.windows.net",
                "kusto.query.url":"https://private-jrsadx.eastus2.kusto.windows.net",
                "aad.auth.appid":"",
                "aad.auth.appkey":"",
                "kusto.sink.tempdir":"/var/tmp/",
                "kusto.sink.flush_size":"10000000",
                "topics": "json_movie_ratings",
                "tasks.max": 8
                }
        }
```
### Execute the creation:
curl -X POST http://localhost:8084/connectors -H "Content-Type: application/json" -d @simple-json-kusto-sink-movies-01.json 

![pic](https://github.com/javierromancsa/images/blob/main/adx-kusto-sink-04.png)

### Edit config to enable log and dead letter queues
```
"errors.log.enable" : "true",
"behavior.on.error" :"log",
"errors.deadletterqueue.topic.name":"dlq_json_movie_ratings",
"errors.log.include.messages": "true",
```
### Also we need to changes the key converter, because record/movie Spiderman has a String with "(" which break the format:
```
"key.converter": "org.apache.kafka.connect.storage.StringConverter" ,
"key.converter.schemas.enable" : "false",
```
## In KsqlDB we need to Create the new table with the json topic so ADX can Sink:
```
CREATE TABLE tbl_movie_ratings2 WITH (KAFKA_TOPIC='json_movie_ratings', VALUE_FORMAT='JSON') AS SELECT m.title, AVG(r.rating) AS avg_ratings, SUM(r.rating) AS sum_rating FROM ratings r LEFT OUTER JOIN tbl_movies m ON m.movie_id = r.movie_id GROUP BY m.title ;
```
### verify the topic key and value:
Ksqldb: print json_movie_ratings from beginning limit 5;
![pic](https://github.com/javierromancsa/images/blob/main/adx-kusto-sink-05.png)

~/confluent-5.5.0/bin/kafka-console-consumer --topic json_movie_ratings --from-beginning --bootstrap-server $kafkabrokers --property print.key=true --property key.separator=" : " --max-messages 5
![pic](https://github.com/javierromancsa/images/blob/main/adx-kusto-sink-06.png)

### Go to ADX and query table:
movies_ratings_kafka_hdi
| where TITLE has "Once Upon a Time in the West"

movies_ratings_kafka_hdi
| count 

## Now let's work on the other table with the avro converter and the schema registry:

### Create the Helm for ADX kafka connector:
cp -R ~/cp-helm-charts/charts/cp-kafka-connect/ myhelmcharts/adx-cp-kafka-connect/
### Edit the replica count based on the partition in the topic:
```
replicaCount: 8
```
### Edit the image that is going to be pull:
```
image: jrsacr.azurecr.io/myadx-kafka-connect
imageTag: v1
```
### Edit the kafka connect properties on this section for the workers:
```
configurationOverrides:
"plugin.path": "/usr/share/java,/usr/share/confluent-hub-components"
"key.converter": "org.apache.kafka.connect.json.JsonConverter"
"value.converter": "org.apache.kafka.connect.json.JsonConverter"
"key.converter.schemas.enable": "false"
"value.converter.schemas.enable": "false"
"internal.key.converter": "org.apache.kafka.connect.json.JsonConverter"
"internal.value.converter": "org.apache.kafka.connect.json.JsonConverter"
"config.storage.replication.factor": "3"
"offset.storage.replication.factor": "3"
"status.storage.replication.factor": "3"
"connector.client.config.override.policy" : "All'
```
### Note: For simplicity I will use the JSON converter but the connector support others. 
Check the link https://github.com/Azure/kafka-sink-azure-kusto#37--kafka-connect-converters

### For further adjusting the values of the values.YAML on the helm config use this link
https://github.com/confluentinc/cp-helm-charts/tree/master/charts/cp-kafka-connect#kafka-connect-deployment

### For now don't enable prometheus as below:
```
prometheus:
  ## JMX Exporter Configuration
  ## ref: https://github.com/prometheus/jmx_exporter
  jmx:
    enabled: false
```
### Get kafka brokers hostnames and add them to the kafka section in value.yml as below:
kafka.bootstrapServers = "wn0-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092,…"

### Optional way to the get this:
```
export clusterName=
export password=
export KAFKABROKERS=$(curl -sS -u admin:$password -G https://$clusterName.azurehdinsight.net/api/v1/clusters/$clusterName/services/KAFKA/components/KAFKA_BROKER | jq -r '["\(.host_components[].HostRoles.host_name):9092"] | join(",")' | cut -d',' -f1,2);
```
### Add Confluent schema registry :
```
cp-schema-registry:
  url: "http://mytest02-cp-schema-registry:8081"
```
### Helm deploy kafka connector
helm install kusto-sink-02 myhelmcharts/adx-cp-kafka-connect/

### start forwarding and check the plugin is available:
sudo nohup kubectl port-forward svc/kusto-sink-02-cp-kafka-connect 8085:8083 &

### Create the kusto sink connector:
### This is an example file:
```
{

	"name": "KustoSinkConnector02",

	"config": {

		"connector.class": "com.microsoft.azure.kusto.kafka.connect.sink.KustoSinkConnector",

		"kusto.sink.flush_interval_ms ": "10000 ",
		"errors.log.enable": "true",
		"behavior.on.error": "fail",
		"key.converter": "org.apache.kafka.connect.storage.StringConverter",
		"value.converter": "io.confluent.connect.avro.AvroConverter",
		"value.converter.schema.registry.url": "http://mytest02-cp-schema-registry:8081",
		"errors.deadletterqueue.topic.name": "dlq_movie_ratings",
		"errors.log.include.messages": "true",
		"kusto.tables.topics.mapping": "[{'topic': 'TBL_MOVIE_RATINGS','db': 'movies', 'table': 'movies_ratings','format': 'avro', 'mapping': 'movies_ratings_mapping'}]",
		"aad.auth.authority": "",
		"kusto.ingestion.url": "https://private-ingest-jrsadx.eastus2.kusto.windows.net",
		"kusto.query.url": "https://private-jrsadx.eastus2.kusto.windows.net",
		"aad.auth.appid": "",
		"aad.auth.appkey": "",
		"kusto.sink.tempdir": "/var/tmp/",
		"kusto.sink.flush_size": "10000000",
		"topics": "TBL_MOVIE_RATINGS",
		"tasks.max": 8
	}

}
```
### Execute the creation:
curl -X POST http://localhost:8085/connectors -H "Content-Type: application/json" -d @simple-avro-kusto-sink-movies-01.json 

### Go to ADX and query table:
movies_ratings
| where TITLE has "Once Upon a Time in the West"

![pic](https://github.com/javierromancsa/images/blob/main/adx-kusto-sink-07.png)

