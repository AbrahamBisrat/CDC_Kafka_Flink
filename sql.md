#### Start MySQL, PostgreSQL clients
```bash
mysql -u root -p123456

psql -h localhost -U postgres
```
```shell
OR from `docker-compose`:
docker compose exec mysql mysql -uroot -p123456
docker compose exec postgres psql -h localhost -U postgres
docker compose exec jobmanager bin/sql-client.sh embedded
```

#### Start SQL client connected to the flink cluster: run from `JobManager`
```bash
bin/sql-client.sh embedded
```

### ___`MySQL`___
```sql
CREATE DATABASE mydb;
USE mydb;
```
```sql
CREATE TABLE products (
    id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description VARCHAR(512)
) AUTO_INCREMENT=101;
```

```sql
INSERT INTO products
VALUES (default, "Scooter", "Small 2-wheel scooter"),
       (default, "car battery", "12V car battery"),
       (default, "12-pack drill bits", "12-pack of drill bits"),
       (default, "Hammer", "12oz hammer"),
       (default, "Hammer", "14oz hammer"),
       (default, "Hammer", "16oz hammer"),
       (default, "rocks", "a box of assorted rocks"),
       (default, "jacket", "waterproof jacket"),
       (default, "spare tire", "spare tire");
```

```sql
CREATE TABLE orders (
    order_id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY,
    order_date DATETIME NOT NULL,
    customer_name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 5) NOT NULL,
    product_id INTEGER NOT NULL,
    order_status BOOLEAN NOT NULL
) AUTO_INCREMENT=10001;
```

```sql
INSERT INTO orders
VALUES (default, "2023-01-01 10:00:00", "John Doe", 19.99, 101, true),
       (default, "2023-01-02 11:00:00", "Jane Smith", 29.99, 102, false),
       (default, "2023-01-04 13:00:00", "Bob Brown", 49.99, 104, false);
```
<br />

---

### ___`PostgreSQL`___
```sql
CREATE TABLE shipments (
    shipment_id SERIAL NOT NULL PRIMARY KEY,
    order_id SERIAL NOT NULL,
    origin VARCHAR(255) NOT NULL,
    destination VARCHAR(255) NOT NULL,
    has_arrived BOOLEAN NOT NULL
);
ALTER SEQUENCE public.shipments_shipment_id_seq RESTART WITH 1001;
```

_To see tables in postgres_
```sql
select * from pg_catalog.pg_tables;
drop table table_name; -- unwanted default table removal if needed.
```

`For capturing changes in the shipments table, we need to set the REPLICA IDENTITY to FULL.`
`OR you will lose track some of the columns in the table.`
```sql
ALTER TABLE public.shipments REPLICA IDENTITY FULL;
```
```sql
INSERT INTO shipments (
    shipment_id,
    order_id,
    origin,
    destination,
    has_arrived
)
VALUES (default, 10001, 'New York', 'Los Angeles', true),
       (default, 10002, 'Chicago', 'Houston', false),
       (default, 10003, 'San Francisco', 'Seattle', true);
```
<br />

---

### ___`Flink SQL`___
```sql
CREATE TABLE products (
    id INT NOT NULL,
    name STRING,
    description STRING
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'mysql',
    'port' = '3306',
    'username' = 'root',
    'password' = '123456',
    'database-name' = 'mydb',
    'table-name' = 'products',
    'scan.incremental.snapshot.enabled' = 'true',
    'scan.incremental.snapshot.chunk.key-column' = 'id'
);
```
```sql
CREATE TABLE orders (
    order_id INT,
    order_date TIMESTAMP(0),
    customer_name STRING,
    price DECIMAL(10, 5),
    product_id INT,
    order_status BOOLEAN
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'localhost',
    'port' = '3306',
    'username' = 'root',
    'password' = '123456',
    'database-name' = 'mydb',
    'table-name' = 'orders',
    'scan.incremental.snapshot.enabled' = 'true',
    'scan.incremental.snapshot.chunk.key-column' = 'order_id'
);
```
```sql
CREATE TABLE shipments (
    shipment_id INT,
    order_id INT,
    origin STRING,
    destination STRING,
    has_arrived BOOLEAN
) WITH (
    'connector' = 'postgres-cdc',
    'hostname' = '127.0.0.1',
    'port' = '5432',
    'username' = 'postgres',
    'password' = 'postgres',
    'database-name' = 'postgres',
    'schema-name' = 'public',
    'table-name' = 'shipments',
    'slot.name' = 'flink_slot'
);
```

```sql
CREATE TABLE enriched_orders (
    order_id INT,
    order_date TIMESTAMP(0),
    customer_name STRING,
    price DECIMAL(10, 5),
    product_id INT,
    order_status BOOLEAN,
    product_name STRING,
    product_description STRING,
    shipment_id INT,
    origin STRING,
    destination STRING,
    has_arrived BOOLEAN,
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector' = 'elasticsearch-6',
    'hosts' = 'http://elasticsearch:9200',
    'index' = 'enriched_orders',
    'format' = 'json',
    'document-type' = '_doc'
);
```

`This combined view of orders, products, and shipments will be used to enrich the data.`
`Will submit the job to the Flink cluster.`
```sql
INSERT INTO enriched_orders
SELECT o.*, p.name, p.description, s.shipment_id, s.origin, s.destination, s.has_arrived
FROM orders AS o
LEFT JOIN products AS p ON o.product_id = p.id
LEFT JOIN shipments AS s ON o.order_id = s.order_id;
```

```
Topology: In this JOB we have 3 sources and 1 sink.
    - Source: MySQL CDC (products, orders)
    - Source: PostgreSQL CDC (shipments)
    - Sink: Elasticsearch (enriched_orders)
    - Enrichment: Join the 3 tables

The CDC connectors will consume all the changes from the source tables.
The CDC connector will run into phases.
1. Snapshot phase: READ all records existing on the DB table & push them downstream to FLINK.
2. Bin Lock READING phase: CDC connector will monitor the log file of the database and generate new records and push them downstream to FLINK.
3. This records are joined together in a JOINT table.
4. This JOINT table will emit UPSERT event to ELASTICSEARCH.
    (This will promise that the data in Elasticsearch is identical to the materialized view of the JOINT table.)
5. Checkpointing: The connector will store the last offset of the log file in the checkpoint periodically.
```


<br />

---

```sql
-- MySQL
-- This will be sent to elasticsearch but the shipment data in it will be null.
INSERT INTO orders
VALUES (default, "2025-03-30 13:20:00", "Boiled Potatos", 108.18, 104, false);
```

```sql
-- PostgreSQL
-- Now that we have a new shipment, this will be sent to elasticsearch & update the record.
INSERT INTO shipments
VALUES (default, 10004, "Los Angeles", "Miami", false);
```

```sql
-- MySQL
-- update order status & observe the propogation of the change to elasticsearch.
UPDATE orders SET order_status = true WHERE order_id = 10004;
```

```sql
-- PostgreSQL
-- update shipment status & observe the propogation of the change to elasticsearch.
UPDATE shipments SET has_arrived = true WHERE shipment_id = 1004;
```

```sql
-- MySQL
-- delete order & observe the propogation of the change to elasticsearch.
DELETE FROM orders WHERE order_id = 10004;
```

<br />

---

### ___`Flink CDC -> Kafka via changelog-json format`___
```sql
-- flink sql
CREATE TABLE kafka_gmv (
    day_str STRING,
    gmv DECIMAL(10, 5),
) WITH (
    'connector' = 'kafka',
    'topic' = 'kafka_gmv',
    'properties.bootstrap.servers' = 'localhost:9092',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'localhost:9092',
    'format' = 'changelog-json',
);
```
```sql
-- flink sql
INSERT INTO kafka_gmv
SELECT DATE_FORMAT(order_date, 'yyyy-MM-dd') AS day_str, SUM(price) AS gmv
FROM orders
GROUP BY DATE_FORMAT(order_date, 'yyyy-MM-dd');
```

Use `kafkacat/kcat` to check if there is any record in kafka_gmv topic.\
[Install from Github here](https://github.com/edenhill/kcat)\
`kafkacat -b localhost:9092 -C -t kafka_gmv`

```sql
-- MySQL
UPDATE orders SET order_status = true WHERE order_id = 10001;
UPDATE orders SET order_status = true WHERE order_id = 10002;
UPDATE orders SET order_status = true WHERE order_id = 10003;

INSERT VALUES INTO orders
VALUES (default, "2025-03-30 13:20:00", "Boiled Potatos", 108.18, 104, true);

UPDATE orders SET price = 50.00 WHERE order_id = 10005;

DELETE FROM orders WHERE order_id = 10005;

-- This should give the same value as the kafka_gmv flink table
SELECT DATE_FORMAT(order_date, 'yyyy-MM-dd') AS day_str, SUM(price) AS gmv
FROM orders
WHERE order_status = true
GROUP BY DATE_FORMAT(order_date, 'yyyy-MM-dd');
```