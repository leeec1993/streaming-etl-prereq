# <div align="center">Getting Started with Stream Processing</div>
## <div align="center">Setup Guide</div>

***

## **Architecture Diagram**

<div align="center">
    <img src="images/architecture.png" width=75% height=75%>
</div>

This setup guide will be walking you through the setup for the [Getting Started with Stream Processing](https://github.com/confluentinc/commercial-workshops/tree/master/series-getting-started-with-cc/workshop-streaming-etl) workshop.

***

## **Prerequisites**

1. Have [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) installed and git clone this repo
```
git clone https://github.com/leeec1993/streaming-etl-prereq.git
```

1. Create a [Confluent Cloud account](https://www.confluent.io/get-started)

1. Create a [MongoDB account](https://www.mongodb.com/try)

1. Create a [GCP account](https://cloud.google.com/free)

1. Have [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) installed

1. [Install MongoDB Database Tools](https://docs.mongodb.com/database-tools/installation/installation/)

***

## <a name="step-1"></a>Step 1: Create a MongoDB Database

1. Login to [MongoDB](https://account.mongodb.com/account/login)

1. Create a Shared (free) Cluster by following the steps [here](https://docs.atlas.mongodb.com/tutorial/deploy-free-tier-cluster/)

***

## <a name="step-1"></a>Step 2: Create a Confluent Cloud Cluster

1. Login to [Confluent Cloud](confluent.cloud)

1. Create a Basic Cluster within the **same cloud and region as your MongoDB Atlas** instance by following the steps [here](https://docs.confluent.io/cloud/current/clusters/create-cluster.html)

1. Within the Basic Cluster [create an API Key Pair](https://docs.confluent.io/cloud/current/access-management/authenticate/api-keys/api-keys.html#create-ak-api-keys-and-associate-with-existing-user-account) with Global access

1. Save the API Key & Secret along with bootstrap server url of your cluster as they will be required later
> **Note:** You can find the bootstrap server url of your cluster by going to the Cluster Overview tab and then Cluster Settings

1. [Enable Schema Registry](https://docs.confluent.io/cloud/current/get-started/schema-registry.html#enable-sr-for-ccloud)

1. [Create an API Key Pair for Schema Registry](https://docs.confluent.io/cloud/current/get-started/schema-registry.html#create-an-api-key-for-ccloud-sr)

1. Save the Schema Registry API Key & Secret along with the API Endpoint as they will be required later

1. [Create two topics](https://docs.confluent.io/cloud/current/client-apps/topics/manage.html#create-a-topic), `clicks` & `transactions`, with 1 partition each

***

## <a name="step-1"></a>Step 3: Generate Dummy Data

We will be using local Datagen connectors to populate the Dummy data that will used for the workshop

1. Create a `.env` file that includes the Confluent Cloud credentials in it - see `.env.example` for a template.

1. Create the local connect instance
```
docker-compose up -d
```
1. Check that your connect instance is up and running
```
docker exec -it kafka-connect bash -c 'echo -e "\n\n  Waiting for Kafka Connect to be available\n"; while : ; do curl_status=$(curl -s -o /dev/null -w %{http_code} http://localhost:8083/connectors) ; echo -e $(date) " Kafka Connect HTTP state: " $curl_status " (waiting for 200)" ; if [ $curl_status -eq 200 ] ; then  break ; fi ; sleep 5 ; done '
curl -s localhost:8083/connector-plugins|jq '.[].class'
```
```
Waiting for Kafka Connect to be available
Thu Sep 23 04:55:14 UTC 2021  Kafka Connect HTTP state:  000  (waiting for 200)
Thu Sep 23 04:55:19 UTC 2021  Kafka Connect HTTP state:  000  (waiting for 200)
Thu Sep 23 04:55:24 UTC 2021  Kafka Connect HTTP state:  000  (waiting for 200)
Thu Sep 23 04:55:30 UTC 2021  Kafka Connect HTTP state:  404  (waiting for 200)
Thu Sep 23 04:55:35 UTC 2021  Kafka Connect HTTP state:  200  (waiting for 200)
```

1. Create the DataGen Connectors
```
curl -i -X PUT -H  "Content-Type:application/json" \
    http://localhost:8083/connectors/datagen-abc-clicks/config \
    -d '{
      "connector.class"               : "io.confluent.kafka.connect.datagen.DatagenConnector",
      "kafka.topic"                   : "clicks",
      "schema.filename"               : "/data/datagen/clicks.avsc",
      "key.converter"                 : "org.apache.kafka.connect.storage.StringConverter",
      "value.converter"               : "org.apache.kafka.connect.json.JsonConverter",
      "value.converter.schemas.enable": "false",
      "max.interval"                  : 10000,
      "iterations"                    : 10000000,
      "tasks.max"                     : "1",
      "transforms"                    : "insertTSNow",
      "transforms.insertTSNow.type"   : "com.github.jcustenborder.kafka.connect.transform.common.TimestampNowField$Value",
      "transforms.insertTSNow.fields" : "click_ts"
  }'
```
```
curl -i -X PUT -H  "Content-Type:application/json" \
    http://localhost:8083/connectors/datagen-abc-transactions/config \
    -d '{
      "connector.class"               : "io.confluent.kafka.connect.datagen.DatagenConnector",
      "kafka.topic"                   : "transactions",
      "schema.filename"               : "/data/datagen/txn.avsc",
      "key.converter"                 : "org.apache.kafka.connect.storage.StringConverter",
      "value.converter"               : "org.apache.kafka.connect.json.JsonConverter",
      "value.converter.schemas.enable": "false",
      "max.interval"                  : 10000,
      "iterations"                    : 10000000,
      "tasks.max"                     : "1",
      "transforms"                    : "insertTSNow",
      "transforms.insertTSNow.type"   : "com.github.jcustenborder.kafka.connect.transform.common.TimestampNowField$Value",
      "transforms.insertTSNow.fields" : "txn_ts"
      }'
```

1. Check that both connectors are up and running
```
curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
         jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
         column -s : -t| sed 's/\"//g'| sort
```
```
source  |  datagen-abc-clicks        |  RUNNING  |  RUNNING  |  io.confluent.kafka.connect.datagen.DatagenConnector
source  |  datagen-abc-transactions  |  RUNNING  |  RUNNING  |  io.confluent.kafka.connect.datagen.DatagenConnector
```
1.  Import Products Data directly into your MonogoDB database
```
mongoimport --uri mongodb+srv://dbUser:MONGODB_PW@MONGODB_ENDPOINT/abc \
            --collection inventory \
            --drop --jsonArray \
            --file data/mongodb/products.json
```
```
2021-04-07T16:00:29.422+0100    connected to: mongodb+srv://[**REDACTED**]@MONGODB_ENDPOINT
2021-04-07T16:00:29.458+0100    dropping: test.inventory
2021-04-07T16:00:29.561+0100    3 document(s) imported successfully. 0 document(s) failed to import.
```

1. Check within your MongoDB Atlas UI that a new **demo** database was created with an **inventory** collection populated with dummy data

***

## <a name="step-1"></a>Step 4: Create a Sink MongoDB Connector

1. Create a MongoDB Atlas sink connector that will automatically create collections and populate the collections with the data residing within your `clicks` and `transactions` topics within Confluent Cloud. From the Confluent Cloud UI, click on the Connectors tab on the navigation menu and select **+Add connector**. Search and click on the MongoDB Atlas Sink icon.

1. Enter the following configuration details. The remaining fields can be left blank.

<div align="center">

| Setting            | Value                        |
|------------------------|-----------------------------------------|
| Topics      | `clicks` `transactions` |
| Name              | MongoDbAtlasSinkConnector                 |
| Input message format           | JSON            |
| Kafka API Key    | From step 1          |
| Kafka API Secret | From step 1             |
| Connection host    | (i.e. cluster0.abc123.mongodb.net)          |
| Connection user    | dbUser         |
| Connection password    | MONGODB_PW          |
| Database name    | demo         |
| Tasks    | 1             |

</div>

1. Once the connector is up and running check that you see the new collections within your MongoDB Atlas database with dummy data populated

***

## <a name="step-1"></a>Step 5: Create a BigQuery Table

1. Go to BigQuery within your GCP console [here](https://console.cloud.google.com/bigquery?_ga=2.241204674.1935688351.1632341687-286332623.1612459805)

2. Create a dataset (i.e. etl_workshop) to be used for the workshop by following the instructions [here](https://cloud.google.com/bigquery/docs/datasets#create-dataset)

***

You are all set to go!  Proceed with the [workshop guide](https://github.com/confluentinc/commercial-workshops/tree/master/series-getting-started-with-cc/workshop-streaming-etl)
