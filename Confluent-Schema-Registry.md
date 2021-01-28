## Confluent Schema Registry
It provides a RESTful interface for storing and retrieving your Avro, JSON Schema, and Protobuf schemas and serializers that plug into Apache Kafka clients that handle schema storage and retrieval for Kafka messages that are sent in any of the supported formats.

### Configuring Kafka schema registry:
cp -R ~/cp-helm-charts/charts/cp-schema-registry/ myhelmcharts/

### Change the service.yaml for the cp-schema-registry helm config to use internal Azure internal loadbalancer
```json
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"

spec:
  type: LoadBalancer
```
### Change/add in values.yaml the "kafka.bootstrapServers":
```
kafka.bootstrapServers = "wn0-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092,…"
```
### Other Properties:
"schema.compatibility.level default" is backward

Is the host.name parameter set? Yes, It should be set when you're running Schema Registry with multiple nodes. Value is advertised in ZooKeeper. In this helm config is the cluster IP. 

### For further changes to the config you use these the following links:
- https://github.com/confluentinc/cp-helm-charts/tree/master/charts/cp-schema-registry#configuration
- https://docs.confluent.io/platform/current/schema-registry/installation/config.html
- https://docs.confluent.io/platform/current/schema-registry/installation/deployment.html#running-sr-in-production

### Change/add in values.yaml the replicaCount to at least 2 based on info below:
"A simple setup with just a few nodes means Schema Registry can fail over easily with a simple multi-node deployment and single primary election protocol." For leader election, use kafkastore.bootstrap.servers instead of kafkastore.connection.url."

If the Schema Registry Security Plugin is installed and configured to use ACLs, it must connect to ZooKeeper and will use kafkastore.connection.url to do so.

You can, and should, have multiple instances with master.eligibility=true. This doesn't mean you'll have multiple masters, it's just a flag to control whether that instance can become master. The case where you'll want to turn this off is if you have one data center that has your primary cluster, but you want to replicate the same schema data to another data center. In that case, the instances in the secondary data center should use master.eligibility=false since they are only intended to be mirrors.

### Some key design decisions:
- Assigns globally unique ID to each registered schema. Allocated IDs are guaranteed to be monotonically increasing but not necessarily consecutive.
- Kafka provides the durable backend, and functions as a write-ahead changelog for the state of Schema Registry and the schemas it contains.
- Schema Registry is designed to be distributed, with single-primary architecture, and ZooKeeper/Kafka coordinates primary election (based on the configuration).

### deploying schema registry
helm install mytest02 myhelmcharts/cp-schema-registry/

### glanced of the logs at one of the pods:
```
HOSTNAME=mytest02-cp-schema-registry-7975bb8466-nrrbt
SCHEMA_REGISTRY_GROUPID=schema01
SCHEMA_REGISTRY_HEAP_OPTS=-Xms512M -Xmx512M
SCHEMA_REGISTRY_HOST_NAME=10.5.4.103
SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS=wn2-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092,wn1-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092,wn0-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092
SCHEMA_REGISTRY_KAFKASTORE_GROUP_ID=mytest02
SCHEMA_REGISTRY_LISTENERS=http://0.0.0.0:8081
SCHEMA_REGISTRY_MASTER_ELIGIBILITY=true
```
### checking schema registry properties to validate
```
kubectl exec -it mytest02-cp-schema-registry-7975bb8466-nrrbt -- sh
cat /etc/schema-registry/schema-registry.properties
kafkastore.group.id=mytest02
master.eligibility=true
heap.opts=-Xms512M -Xmx512M
kafkastore.bootstrap.servers=wn2-jrs02k.qz1stakpznlepcjn2bzi2treqb.cx.internal.cloudapp.net:9092,..
listeners=http://0.0.0.0:8081
host.name=10.5.4.103
groupid=schema01
```
### Add the name of the schema registry k8s service on the values.yaml of the cp-kafka-connect helm.
```
cp-schema-registry:
url: "http://mytest02-cp-schema-registry:8081"
```
### deploy the new kafka connect:
helm install mytest03 myhelmcharts/cp-kafka-connnect/

### Create a file simple-jdbc-source-bulk-movies-01.json with the following: 
```json
{
        "name": "jdbc_source_postgres_01",
        "config": {
                "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
                "connection.url": "jdbc:postgresql://postgres2:5432/postgresdb",
                "connection.user": "",
                "connection.password": "",
                "topic.prefix": "tblmovies01-",
                "tasks.max": 1,
                "mode": "bulk",
                "validate.non.null": "false",
                "table.whitelist": "movies",
                "table.poll.interval.ms": 60000

                }
}
```
### Start the port-forwarding:
sudo nohup kubectl port-forward svc/mytest01-cp-kafka-connect 803:8083 &

### Create the new connector
curl -X POST http://localhost:803/connectors -H "Content-Type: application/json" -d @myconnectors/simple-jdbc-source-bulk-movies-01.json | json_pp

![image](https://github.com/javierromancsa/images/blob/main/images/cp-sch-reg-01.jpg)

### let's inspect the _schemas topic:
![image](~/images/cp-sch-reg-02.png)
### Why is empty? Because of the way we configure the kafka connector properties:
```
"key.converter": "org.apache.kafka.connect.json.JsonConverter"
"value.converter": "org.apache.kafka.connect.json.JsonConverter"
```
### No avro , protobuf or jsonschema converter.
More details in this link https://docs.confluent.io/platform/current/connect/userguide.html#configuring-key-and-value-converters

### lets create a new connector with avro converter by adding the 4 lines of code to a copy of the previous connector config:
```
  "key.converter": "io.confluent.connect.avro.AvroConverter",
  "key.converter.schema.registry.url": "http://mytest02-cp-schema-registry:8081",
  "value.converter": "io.confluent.connect.avro.AvroConverter",
  "value.converter.schema.registry.url": "http://mytest02-cp-schema-registry:8081",
```
### Create the new connector:
curl -X POST http://localhost:803/connectors -H "Content-Type: application/json" -d @myconnectors/simple-avro-jdbc-source-bulk-movies-01.json | json_pp
![image](../images/cp-sch-reg-03.jpg)

### Let's inspect the topic "_schema" again:
confluent-5.5.0/bin/kafka-console-consumer --topic _schemas --bootstrap-server $kafkabrokers --from-beginning --property print.key=true --property key.separator=" : "
[image](../images/cp-sch-reg-04.png)

### Now inspect the new topic and the previous one:
```
confluent-5.5.0/bin/kafka-avro-console-consumer --topic avro_tblmovies01-movies --bootstrap-server $kafkabrokers --property schema.registry.url=http://10.5.5.82:8081 --property print.key=true --property key.separator=" : " --max-messages 1
```
![image](../images/cp-sch-reg-05.png)

