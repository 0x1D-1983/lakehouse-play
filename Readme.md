# Lakehouse Demo using Iceberg, Polaris, Trino and MinIO

![Architecture Diagram](chart.png)

## Services
 - Trino Web UI: http://localhost:8080
 - MinIO UI: http://localhost:9001 (admin/password)
 - MinIO API: http://localhost:9000
 - Polaris: http://localhost:8181
 - Flink Web UI: http://localhost:8081

## Create an Iceberg catalog

Get access token
``` shell
ACCESS_TOKEN=$(curl -X POST \
  http://localhost:8181/api/catalog/v1/oauth/tokens \
  -d 'grant_type=client_credentials&client_id=root&client_secret=secret&scope=PRINCIPAL_ROLE:ALL' \
  | jq -r '.access_token')
```

Create an Iceberg catalog in Polaris
``` shell
  curl -i -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  http://localhost:8181/api/management/v1/catalogs \
  --json '{
    "name": "polariscatalog",
    "type": "INTERNAL",
    "properties": {
      "default-base-location": "s3://warehouse",
      "s3.endpoint": "http://minio:9000",
      "s3.path-style-access": "true",
      "s3.access-key-id": "admin",
      "s3.secret-access-key": "password",
      "s3.region": "dummy-region"
    },
    "storageConfigInfo": {
      "roleArn": "arn:aws:iam::000000000000:role/minio-polaris-role",
      "storageType": "S3",
      "allowedLocations": [
        "s3://warehouse/*"
      ]
    }
  }'
```

Check that the catalog was correctly created in Polaris:
``` shell
curl -X GET http://localhost:8181/api/management/v1/catalogs \
  -H "Authorization: Bearer $ACCESS_TOKEN" | jq
```

## Set Up Permissions

Create a catalog admin role
``` shell
curl -X PUT http://localhost:8181/api/management/v1/catalogs/polariscatalog/catalog-roles/catalog_admin/grants \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  --json '{"grant":{"type":"catalog", "privilege":"CATALOG_MANAGE_CONTENT"}}'
```

Create a data engineer role
``` shell
curl -X POST http://localhost:8181/api/management/v1/principal-roles \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  --json '{"principalRole":{"name":"data_engineer"}}'
```

Connect the roles
``` shell
curl -X PUT http://localhost:8181/api/management/v1/principal-roles/data_engineer/catalog-roles/polariscatalog \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  --json '{"catalogRole":{"name":"catalog_admin"}}'
```

Give root the data engineer role
``` shell
curl -X PUT http://localhost:8181/api/management/v1/principals/root/principal-roles \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  --json '{"principalRole": {"name":"data_engineer"}}'
```

Check that the role was correctly assigned to the root principal:
``` shell
curl -X GET http://localhost:8181/api/management/v1/principals/root/principal-roles -H "Authorization: Bearer $ACCESS_TOKEN" | jq
```

## Create an Iceberg table and run some queries

Open Trino session
``` shell
docker exec -it trino trino --server localhost:8080 --catalog iceberg
```

Create a schema first (a namespace in Polaris)
``` sql
CREATE SCHEMA db;
```

Activate the schema
``` sql
USE db;
```

Create a table:
``` sql
CREATE TABLE customers (
  customer_id BIGINT,
  first_name VARCHAR,
  last_name VARCHAR,
  email VARCHAR
);
```

Insert a few records in that table:
``` sql
INSERT INTO customers (customer_id, first_name, last_name, email) 
VALUES (1, 'Rey', 'Skywalker', 'rey@rebelscum.org'),
       (2, 'Hermione', 'Granger', 'hermione@hogwarts.edu'),
       (3, 'Tony', 'Stark', 'tony@starkindustries.com');
```

Query the table
``` sql
SELECT * FROM customers;
```

Update record
``` sql
UPDATE customers
SET last_name = 'Granger-Weasley'
WHERE customer_id = 2;
```

List table history
``` sql
SELECT snapshot_id, committed_at, summary
FROM "customers$snapshots"
ORDER BY committed_at DESC;
```

Query the table with time travel
``` sql
SELECT * FROM customers FOR TIMESTAMP AS OF TIMESTAMP '2025-08-05 09:53:43.994 UTC';
```

## Configuring the Streaming Pipeline

Start Flink SQL
```
docker exec -it flink-sql-client sql-client.sh
```

### Setting up the Kafka - Flink pipeline
``` sql
CREATE CATALOG kafka_catalog WITH ('type'='generic_in_memory');
  
CREATE DATABASE kafka_catalog.sales_db;
  
CREATE TABLE kafka_catalog.sales_db.transactions (
    transaction_id   STRING,  
    user_id          STRING,  
    amount           DECIMAL(10, 2),  
    currency         STRING,  
    merchant         STRING,  
    transaction_time TIMESTAMP(3),
    WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND
) WITH (  
    'connector' = 'kafka',  
    'properties.bootstrap.servers' = 'broker:29092',  
    'format' = 'json',  
    'scan.startup.mode' = 'earliest-offset',  
    'topic' = 'transactions',
    'value.format' = 'json-registry'
);
```

Quick test it, this should write to the associated Kafka topic
``` sql
INSERT INTO kafka_catalog.sales_db.transactions  
VALUES ('TXN_001', 'USER_123', 45.99, 'GBP', 'Amazon', TIMESTAMP '2025-06-23 10:30:00'),  
       ('TXN_002', 'USER_456', 12.50, 'GBP', 'Starbucks', TIMESTAMP '2025-06-23 10:35:00'),       
       ('TXN_003', 'USER_789', 89.99, 'USD', 'Shell', TIMESTAMP '2025-06-23 10:40:00'),       
       ('TXN_004', 'USER_123', 156.75, 'EUR', 'Tesco', TIMESTAMP '2025-06-23 10:45:00'),       
       ('TXN_005', 'USER_321', 8.99, 'GBP', 'McDonald''s', TIMESTAMP '2025-06-23 10:50:00');
```

You can test that the messages are indeed in the topic
``` shell
kafka-console-consumer --bootstrap-server broker:9092 --topic transactions --from-beginning
```

### Setting up the Flink - Polaris Iceberg catalog pipeline

``` sql
CREATE CATALOG polaris_catalog WITH (
    'type'='iceberg',  
    'catalog-type'='rest',  
    'uri'='http://polaris:8181/api/catalog',  
    'warehouse'='polariscatalog',  
    'oauth2-server-uri'='http://polaris:8181/api/catalog/v1/oauth/tokens',  
    'credential'='root:secret',  
    'scope'='PRINCIPAL_ROLE:ALL'
);
```

Create a database in Flink in that catalog (a database in Flink maps to a namespace in Polaris)
``` sql
CREATE DATABASE polaris_catalog.sales_db;
```

You must enable checkpointing (exactly-once semantics in streaming mode).
``` sql
SET 'execution.checkpointing.interval' = '10s';
```

Create the dynamic table
``` sql
CREATE TABLE polaris_catalog.sales_db.transactions AS  
    SELECT * FROM kafka_catalog.sales_db.transactions;
```

## Genereate mock data with the JR Tool

Create a topic
``` shell
# jr createTopic transactions -p 1 -r 1
kafka-topics --create --bootstrap-server broker:9092 --replication-factor 1 --partitions 1 --topic transactions
```

Generate data
``` shell
jr run \
  --embedded "$(cat transaction.json)" \
  --num 5 \
  --frequency 500ms \
  --output kafka \
  --topic transactions \
  --kafkaConfig kafka/config.properties \
  --schemaRegistry kafka/registry.properties \
  --serializer json-schema \
  --autoRegisterSchemas false
```