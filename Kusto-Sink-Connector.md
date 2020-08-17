# Azure Kusto Sink Connector
The Azure Kusto Sink Connector is used to read records form Kafka topics and ingest them into Kusto Tables.

## Features       


The Azure Kusto Sink Connector offers the following features:     

-  **At-least-Once Semantic**: The connector creates a new entry into the Kusto table for each record in Kafka topic. Duplicates can occur when failure, rescheduling or re-configuration happens. This **semantic** is followed when `behavior.on.error` is set to `fail` mode. In case of `log` and `ignore` modes, the connector promises **at-most-once** semantics.
-  **Automatic Retries**: The Azure Kusto Sink Connector may experience network failures while connecting to the Kusto Ingestion URI. The connector will automatically retry with an exponential backoff to ingest records. The property `errors.retry.max.time.ms`  controls the maximum time until which the connector will retry ingesting the records.
-  **Compression**: The connector batches the records into a file and performs GZIP compression before ingesting file to Kusto. The GZIP compression is applied to all the [data formats](#kusto-record-formats) supported by the Connector.  
---
## Prerequisites

The following are required to run the **Azure Kusto** Sink Connector:

- Kafka Broker: Confluent Platform 3.3.0 or above, or Kafka 0.11.0 or above
- Connect: Confluent Platform 4.0.0 or above, or Kafka 1.0.0 or above
- Java 1.8
- Kusto Ingestion URL
- Database ingestor role
- Configure ingestion `Topics` and `Miscellaneous Dead-Letter Queue Topic` with the desired replication and partitions.
> **Note**   
>If `topics` or `misc.deadletterqueue.topic.name` is not present, the connector will automatically create them with the default 
>replication factor and partition numbers as one.

----

## Ingesting Records to Kusto Table
The Azure Kusto Sink connector consumes records from the specified topics, organizes them into different partitions, writes batches of records in each partition to a file, and then ingest these files to the specified Kusto Table.

### Kusto Record Formats
The Kusto connector can serialize records using a number of formats. The `format` attribute in the connector’s `kusto.tables.topics.mapping` configuration property identifies the format to be used. The Kusto connector comes with several implementations:

- **Avro**: Use `value.converter=io.confluent.connect.avro.AvroConverter`, and `format:'avro` in `kusto.tables.topics.mapping` configuration property  to ingest avro records into kusto table.
- **Schemaless JSON**: Use `value.converter=org.apache.kafka.connect.json.JsonConverter`,`value.converter.schemas.enable=false`, and `format:'json` in `kusto.tables.topics.mapping` configuration property  to ingest json records having no schema into kusto table.
- **JSON with Schema**: Use `value.converter=org.apache.kafka.connect.json.JsonConverter`,`value.converter.schemas.enable=true`, and `format:'json` in `kusto.tables.topics.mapping` configuration property  to ingest json records having a schema into kusto table. Alternatively, use `value.converter=io.confluent.connect.json.JsonSchemaConverter`, and `format:'json` in `kusto.tables.topics.mapping` configuration property  to ingest json records having a schema into kusto table. Note **Confluent Platform 5.5.0** or above required for using the new JsonSchemaConverter.
- **String**: Use `value.converter=org.apache.kafka.connect.storage.StringConverter`, and `format:'avro`, `format:'json`, `format:'csv`, in `kusto.tables.topics.mapping` configuration property  to ingest avro, json, csv records respectively into kusto table.
- **Byte**: Use `value.converter=org.apache.kafka.connect.converters.ByteArrayConverter`, and `format:'avro`, `format:'json`, `format:'csv`, in `kusto.tables.topics.mapping` configuration property  to ingest avro, json, csv records respectively into kusto table.
 
----
## Installation
#### Prerequisite
[Confluent Platform](https://www.confluent.io/download) must be installed.

#### Clone
Clone the [Azure Kusto Sink Connector](https://github.com/Azure/kafka-sink-azure-kusto)
```shell script
git clone git://github.com/Azure/kafka-sink-azure-kusto.git
cd ./kafka-sink-azure-kusto
```
#### Build
##### Requirements

- Java >=1.8
- Maven

Building locally using Maven is simple:

```bash
mvn clean install
```

The connector jar along with jars of associated dependencies will be found under `target/components/packages/microsoftcorporation-kafka-sink-azure-kusto-1.0.1/microsoftcorporation-kafka-sink-azure-kusto-1.0.1/lib/` folder.

Use the following command to build an uber jar.

```bash
mvn clean compile assembly:single
```
:grey_exclamation: Move the jar inside a folder in /share/java folder within the Confluent Platform installation directory.


----
## Configuration Properties
For a complete list of configuration properties for this connector, see [Azure Kusto Sink Connector Configuration Properties](<// Todo add Link>).

---     

## Dead-Letter-Queue Support   
### Error Dead-Letter-Queue     
The Azure Kusto Sink Connector uses the [error dead-letter queue (DLQ)](https://kafka.apache.org/24/documentation.html#sinkconnectconfigs) to produces failure records for messages that result in an error when processed by this sink connector, or its transformations or converters. 
### Miscellaneous Dead-Letter-Queue
The Azure Kusto Sink Connector uses the miscellaneous dead-letter queue (DLQ) to produce failed records into Kafka for the following scenarios: 
- Failed to be ingested in to the Kusto cluster when `IngestionServiceException` is caught due to network interruptions or unavailability of the Kusto Cluster.
- Failed to be written into file for ingestion when `IOException |DataException` are caught due to temp directory does not exist , user has insufficient permission to write to/ create a file in the temp directory.   
> Note    
> When using Dead-Letter Queues in a secured environment add additional security configurations prepended with `error.deadletterqueue.` and `misc.deadletterqueue.` respectively.   
---
## Retries
The Azure Kusto Sink Connector may experience problems writing to the Kusto Cluster, due to network interruptions, or even unavailability of the Kusto Cluster. The connector will retry the for a maximum time duration mentioned as `errors.retry.max.time.ms` with a backoff period between subsequent retries as mentioned in the `errors.retry.backoff.time.ms` before failing. 
> **Important**    
> Since retries occur both at the connector and Kusto Cluster level, they can be for a longer duration within which the Kafka Consumer may leave the group. This will result in a new Consumer
reading records from the last committed offset leading to duplication of records in Kusto Database. Also, if the error persists,
it might also result in duplicate records to be written into the DLQ topic.
Recommendation is to set the following worker configuration as `connector.client.config.override.policy=All` and set the
`consumer.override.max.poll.interval.ms` config to a high enough value to avoid consumer leaving the group while the
Connector is retrying.
---
## Quick Start
The quick start guide uses the Azure Sink Connector to consume records from a Kafka topic and ingest records into Kusto tables.

> **Note**     
> A SAS token is generated after the successful authentication of the connector. Therefore, if a change in permission occur while the connector is running the connector will continue to run without failing until the granted SAS token expires.    

> **Warning**    
> The Kusto ingestion failures are logged at file level. Therefore, a single log entry for records that fail to get ingested via file ingestion. Also, a single malformed record in a file results in failure of ingestion for the entire file.

### Kusto Table and Table Mapping Setup

Before using the Azure Kusto Sink Connector, It is required to set up the table and its corresponding table mapping depending on the record schema, and the converter type.

 Use the following to [create a sample table](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/create-table-command) in Kusto.
 
 ```
.create table SampleKustoTable ( Name:string, Age:int)
```

Since, the Avro Converter is being used for the demo, It is necessary to create an Avro-based table [ingestion mapping](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/create-ingestion-mapping-command) using the following code Snippet.
```
.create table SampleKustoTable ingestion avro mapping "SampleAvroMapping"
'['
'    { "column" : "Name", "datatype" : "string", "Properties":{"Path":"$.Name"}},'
'    { "column" : "Age", "datatype" : "int", "Properties":{"Path":"$.Age"}}'
']'
```
> **Note**   
> Properties are the corresponding fields in the record schema.

For more information about ingestion mapping, see [Kusto Data mappings](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/mappings).

### Start Confluent
Start the Confluent Platform using the following [Confluent CLI](https://docs.confluent.io/current/cli/index.html#cli) command:

 ```shell script
confluent local start
```
> **Important**    
> Do not use the Confluent CLI in production environments. 

### Property-based example
Create a configuration file `kusto-sink.properties` with the following content. This file should
be placed inside the Confluent Platform installation directory. This configuration is used typically along with standalone workers.

```
name=azure-0
connector.class=com.microsoft.azure.kusto.kafka.connect.sink.KustoSinkConnector
topics=SampleAvroTopic
tasks.max=1

kusto.tables.topics.mapping=[{'topic': 'SampleAvroTopic','db': 'DatabaseName', 'table': 'SampleKustoTable','format': 'avro', 'mapping':'SampleAvroMapping'}]

kusto.url=https://ingest-<your cluster URI>.kusto.windows.net
aad.auth.authority=****
aad.auth.appid=****
aad.auth.appkey=****
tempdir.path=/var/temp/

key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=io.confluent.connect.avro.AvroConverter
value.converter.schema.registry.url=http://localhost:8081

behavior.on.error=LOG
misc.deadletterqueue.bootstrap.servers=localhost:9092
misc.deadletterqueue.topic.name=network-error-dlq-topic

errors.tolerance=all
errors.deadletterqueue.topic.name=connect-dlq-topic
errors.deadletterqueue.topic.replication.factor=1
errors.deadletterqueue.context.headers.enable=true
errors.retry.max.time.ms=2000
errors.retry.backoff.time.ms=1000
```

Run the connector with this configuration.
```shell script
confluent local load kusto-sink-connector -- -d kusto-sink.properties
```

The output should resemble:
```json
{
  "name": "kusto-sink-connector",
  "config": {
    "connector.class": "com.microsoft.azure.kusto.kafka.connect.sink.KustoSinkConnector",
    "topics": "SampleAvroTopic",
    "tasks.max": "1",
    "kusto.tables.topics.mapping": "[{'topic': 'SampleAvroTopic','db': 'DatabaseName', 'table': 'SampleKustoTable','format': 'avro', 'mapping':'SampleAvroMapping'}]",
    "kusto.url": "https://ingest-<your cluster URI>.kusto.windows.net",
    "aad.auth.authority": "****",
    "aad.auth.appid": "****",
    "aad.auth.appkey": "****",
    "tempdir.path": "/var/temp/",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://localhost:8081",
    "behavior.on.error": "LOG",
    "misc.deadletterqueue.bootstrap.servers": "localhost:9092",
    "misc.deadletterqueue.topic.name": "network-error-dlq-topic",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "connect-dlq-topic",
    "errors.deadletterqueue.topic.replication.factor": "1",
    "errors.deadletterqueue.context.headers.enable": "true",
    "errors.retry.max.time.ms": "2000",
    "errors.retry.backoff.time.ms": "1000",
    "name": "kusto-sink-connector"
  },
  "tasks": [],
  "type": "sink"
}
```

Confirm that the connector is in a RUNNING state.
```shell script
confluent local status kusto-sink-connector
```

The output should resemble:

```shell script
{
  "name": "kusto-sink-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "127.0.1.1:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "127.0.1.1:8083"
    }
  ],
  "type": "sink"
}
```

### REST-based example
Use this setting with [distributed workers](https://docs.confluent.io/current/connect/concepts.html#distributed-workers). 
Write the following JSON to `config.json`, configure all the required values, and use the following command to post the configuration to one of the distributed connect workers. 
Check here for more information about the Kafka Connect [REST API](https://docs.confluent.io/current/connect/references/restapi.html#connect-userguide-rest)

```json
{
    "name": "KustoSinkConnectorCrimes",
    "config": {
        "connector.class":"com.microsoft.azure.kusto.kafka.connect.sink.KustoSinkConnector",
	"topics":"SampleAvroTopic",
	"tasks.max":1,
	"kusto.tables.topics.mapping":[{"topic": "SampleAvroTopic","db": "DatabaseName", "table": "SampleKustoTable","format": "avro", "mapping":"SampleAvroMapping"}],
	"kusto.url":"https://ingest-azureconnector.centralus.kusto.windows.net",
	"aad.auth.authority":"****",
	"aad.auth.appid":"****",
	"aad.auth.appkey":"****",
	"tempdir.path":"/var/tmp",
	"key.converter":"org.apache.kafka.connect.storage.StringConverter",
	"value.converter":"io.confluent.connect.avro.AvroConverter",
	"value.converter.schema.registry.url":"http://localhost:8081",
	"behavior.on.error":"LOG",
	"misc.deadletterqueue.bootstrap.servers": "localhost:9092",
    	"misc.deadletterqueue.topic.name": "network-error-dlq-topic",
    	"errors.tolerance": "all",
        "errors.deadletterqueue.topic.name": "connect-dlq-topic",
        "errors.deadletterqueue.topic.replication.factor": "1",
        "errors.deadletterqueue.context.headers.enable": "true",
        "errors.retry.max.time.ms": "2000",
        "errors.retry.backoff.time.ms": "1000"
    }
}
```
>**Note**    
>Change the `confluent.topic.bootstrap.servers` property to include your broker address(es) and change the `confluent.topic.replication.factor` to 3 for staging or production use.

Use curl to post a configuration to one of the Kafka Connect workers. Change `http://localhost:8083/` to the endpoint of one of your Kafka Connect worker(s).
```shell script
curl -sS -X POST -H 'Content-Type: application/json' --data @config.json http://localhost:8083/connectors
```
Use the following command to update the configuration of existing connector.
```shell script
curl -s -X PUT -H 'Content-Type: application/json' --data @config.json http://localhost:8083/connectors/kusto-sink-connector/config
```
Confirm that the connector is in a `RUNNING` state by running the following command:
```
curl http://localhost:8083/connectors/pagerduty-sink-connector/status | jq
```

The output should resemble:

```shell script
{
  "name": "kusto-sink-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "127.0.1.1:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "127.0.1.1:8083"
    }
  ],
  "type": "sink"
}
```

To produce Avro data to Kafka topic: `SampleAvroTopic`, use the following command.
```shell script
./bin/kafka-avro-console-producer --broker-list localhost:9092 --topic SampleAvroTopic --property value.schema='{"type":"record","name":"details","fields":[{"name":"Name","type":"string"}, {"name":"Age","type":"int"}]}'
```

While the console is waiting for the input, use the following three records and paste each of them on the console.
```shell script
{"Name":"Alpha Beta", "Age":1}
{"Name":"Beta Charlie", "Age":2}
{"Name":"Charlie Delta", "Age":3}
```

Finally, check the Kusto table `SampleKustoTable` to see the newly ingested records.

---
## Additional Documentation

- [Azure Kusto Sink Connector Configuration Property]()
- [Changelog]()      





