## What is Kafka Connect?
Kafka Connect is a tool for scalably and reliably streaming data between Apache Kafka and other systems. It makes it simple to quickly define connectors that move large collections of data into and out of Kafka. Kafka Connect can ingest entire databases or collect metrics from all your application servers into Kafka topics, making the data available for stream processing with low latency.
## Why Kafka Connect?
Kafka Connect features include:
- A common framework for Kafka connectors - Kafka Connect standardizes integration of other data systems with Kafka, simplifying connector development, deployment, and management
- Distributed and standalone modes - scale up to a large, centrally managed service supporting an entire organization or scale down to development, testing, and small production deployments
- REST interface - submit and manage connectors to your Kafka Connect cluster via an easy to use REST API
- Automatic offset management - with just a little information from connectors, Kafka Connect can manage the offset commit process automatically so connector developers do not need to worry about this error prone part of connector development
- Distributed and scalable by default - Kafka Connect builds on the existing group management protocol. More workers can be added to scale up a Kafka Connect cluster.
- Streaming/batch integration - leveraging Kafka's existing capabilities, Kafka Connect is an ideal solution for bridging streaming and batch data systems

## Getting Ready to use Kafka Connect:
- Deploying a Kafka Cluster
  - HDInsight inside your own Vnet https://docs.microsoft.com/en-us/azure/hdinsight/hdinsight-plan-virtual-network-deployment 
  - Kafka in AKS with Strimzi https://dev.to/azure/kafka-on-kubernetes-the-strimzi-way-part-1-57g7
- Deploying your AKS Cluster into your Vnet and Private zone 
  https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni
  https://docs.microsoft.com/en-us/azure/aks/private-clusters
- Azure Bastion and Jumpbox or Azure VPN with P2S connection
  TBD
- Pre-define Automation deployment 
  TBD
## Deploy Confluent Kafka Connect:

### copy source charts from Confluent
```
git clone https://github.com/confluentinc/cp-helm-charts.git
mkdir myhelmcharts/
cd myhelmcharts
cp -R ~/cp-helm-charts/charts/cp-kafka-connect/ myhelmcharts/
```
### Edit the replica count to match the numbers of tables/files are going to be source/sink or the number of partitions in the topic:
`replicaCount: 8`

### Edit the kafka connect properties on the section settings for the worker:
```
configurationOverrides:
"plugin.path": "/usr/share/java,/usr/share/confluent-hub-components"
"key.converter": "org.apache.kafka.connect.json.JsonConverter"
"value.converter": "org.apache.kafka.connect.json.JsonConverter"
"key.converter.schemas.enable": "false"
"value.converter.schemas.enable": "false"
"config.storage.replication.factor": "3"
"offset.storage.replication.factor": "3"
"status.storage.replication.factor": "3"
"connector.client.config.override.policy" : "All'
```
### For further adjusting the settings of the values.YAML on the helm config use this link:
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
### Deploying the helm and checking if is running:
```
helm install test01 myhelmcharts/cp-kafka-connect/
Kubectl get pods
kubectl logs test01-cp-kafka-connect-5f8cc6fd-2q292|tail
```
### Getting confluent community and use kafka binaries: http://packages.confluent.io/archive/5.5/
```
wget http://packages.confluent.io/archive/5.5/confluent-community-5.5.0-2.12.tar.gz
tar xzf confluent-community-5.5.0-2.12.tar.gz

confluent-5.5.0/bin/kafka-console-consumer --topic test01-cp-kafka-connect-offset --from-beginning --bootstrap-server $kafkabrokers
confluent-5.5.0/bin/kafka-console-consumer --topic test01-cp-kafka-connect-config --from-beginning --bootstrap-server $kafkabrokers

sudo nohup kubectl port-forward svc/test01-cp-kafka-connect 803:8083 &
```
### Validate Connector config before creating:
```
curl -X PUT http://localhost:803/connector-plugins/JdbcSourceConnector/config/validate -H "Content-Type: application/json" -d '{
       "name":"jdbc_source_postgres_01",
       "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
       "connection.url": "jdbc:postgresql://postgres2:5432/postgresdb",
       "connection.user": "",
       "connection.password": "",
       "table.whitelist": "movies",
       "mode": "bulk",
       "tasks.max": "1",
       "topic.prefix": "test01",
       "validate.non.null": "false",
       "table.poll.interval.ms": 60000
}' | json_pp
```
### Create Connector for PostgresSQL using JDBCsourceconnector in bulk mode:
```
curl -X POST http://localhost:803/connectors -H "Content-Type: application/json" -d '{
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
        }' | json_pp
```
### Check the topics
```
confluent-5.5.0/bin/kafka-console-consumer --topic tblmovies01-movies --bootstrap-server $kafkabrokers
confluent-5.5.0/bin/kafka-console-consumer --value-deserializer org.apache.kafka.common.serialization.StringDeserializer --key-deserializer org.apache.kafka.common.serialization.StringDeserializer --topic tblmovies01-movies --bootstrap-server $kafkabrokers
```
### Create Connector for PostgresSQL using JDBCsourceconnector with incrementing mode on 'id' column:
```
curl -X POST http://localhost:803/connectors -H "Content-Type: application/json" -d '{
        "name": "jdbc_source_postgres_02",
        "config": {
                "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
                "connection.url": "jdbc:postgresql://postgres2:5432/postgresdb",
                "connection.user": "",
                "connection.password": "",
                "topic.prefix": "newtblmovies01-",
                "tasks.max": 1,
                "mode": "incrementing",
                "incrementing.column.name": "id",
                "validate.non.null": "false",
                "table.whitelist": "movies",
                "table.poll.interval.ms": 1000

                }
        }'
```
### Check the topics
`confluent-5.5.0/bin/kafka-console-consumer --topic newtblmovies01-movies --bootstrap-server $kafkabrokers`

### insert records to the movies table
```
psql -h 10.5.4.155 -U postgresadmin --password -p 5432 -d postgresdb
=# INSERT INTO movies(id,title,release_year,unknown_1,country,unknown_2,genres,actors,director,composers,screenwriters,cinematographers,production_companies) VALUES (921,'Diego Book',2020,16,'Puerto Rico',18,'Western','Sambrell','Leone','Morricone','Bertolucci','Colli','Paramount Pictures');

=# INSERT INTO movies(id,title,release_year,unknown_1,country,unknown_2,genres,actors,director,composers,screenwriters,cinematographers,production_companies) VALUES (922,'Javier Book',2020,16,'Puerto Rico',18,'Western','Sambrell','Leone','Morricone','Bertolucci','Colli','Paramount Pictures');
```
### checking kafka-connect properties to validate
```
kubectl exec -it test01-cp-kafka-connect-85855d754c-ddcxk -- sh
cat /etc/kafka-connect/kafka-connect.properties
```