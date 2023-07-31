# Pinecone - steps to get dynamodb connector running

kubectl -n confluent create secret generic ccloud-credentials --from-file=plain.txt=ccloud-credentials.txt

ccloud-credentials.txt has API key from confluent cloud
Username is key id
password is secret key


## install confluent for k8s
```
kubectl create ns confluent
helm repo add confluentinc https://packages.confluent.io/helm
helm upgrade --install operator confluentinc/confluent-for-kubernetes -n confluent
kubectl get pods -n confluent
```

## install connect, adding the dynamodb plugin

I had to generate the zip file myself because it needs a manifest file along with the fatjar from trustpilot.
This process can be easily automated.

```
kubectl -n confluent apply -f kafka-connect.yaml
```

kafka-connect.yaml:
```
---
apiVersion: platform.confluent.io/v1beta1
kind: Connect
metadata:
  name: connect
spec:
  replicas: 1
  image:
    application: confluentinc/cp-server-connect:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
  build:
    type: onDemand
    onDemand:
      plugins:
        locationType: url
        url:
          - name: kafka-connect-dynamodb
            archivePath: https://ivan-connector-testing.s3.eu-west-1.amazonaws.com/pinecone-kafka-connect-dynamodb-0.1.zip
            checksum: b4c10e714a69cc0caeab3298a531cfdf1ae461f89720f40bf4f253e566362fe4b9983c1f9b4be82825b974835acffb0288ade649889b0dbf2e8f8850cf83e458
  dependencies:
    kafka:
      bootstrapEndpoint: pkc-z9doz.eu-west-1.aws.confluent.cloud:9092
      authentication:
        type: plain
        jaasConfig:
          secretRef: ccloud-credentials
      tls:
        enabled: true
        ignoreTrustStoreConfig: true 
```

## Create table and kafka topic

Enable streams on the table, and add tags:
- environment=dev
- usekafka

Create a topic in confluent with name: ```dynamodb-<tablename>```

## Start a connector on connect

```
kubectl -n confluent apply -f connector.yaml
```

connector.yaml:
```
---
apiVersion: platform.confluent.io/v1beta1
kind: Connector
metadata:
  name: dynamodbcdc
  namespace: confluent
spec:
  class: "com.trustpilot.connector.dynamodb.DynamoDBSourceConnector"
  taskMax: 1
  connectClusterRef:
    name: connect
  configs:
    aws.region: "eu-west-1"
    aws.access.key.id: "FILL ME IN"
    aws.secret.key: "FILL ME IN"
    dynamodb.table.env.tag.key: "environment"
    dynamodb.table.env.tag.value: "dev"
    dynamodb.table.ingestion.tag.key: "usekafka"
    kcl.table.billing.mode: "PROVISIONED"
    kafka.topic.prefix: "dynamodb-"
```

## Test it out

- Open the messages tab for the topic in confluent.cloud
- Write something to the dynamodb table (can also be done through web interface)
