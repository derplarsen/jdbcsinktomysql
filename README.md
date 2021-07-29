# JDBC Sink to MySQL Example

The ***JDBC Sink Connector*** provided by Confluent is a very powerful and flexible data sink mechanism. 

With it you can do the following, and they are included in this example:

 - Insert a record with data from a topic 
 - Update a record with data from a topic
 - Delete a record with data from a topic

Additionally, there are a couple of features that are possible but not demonstrated in this example:

 - Add new fields in a topic automatically (configurable) 
 - Create a table automatically from a topic in Kafka

## Prerequisites:

1. Download the latest version of Confluent Platform
2. Follow the instructions to start the cluster [https://docs.confluent.io/platform/current/quickstart/ce-quickstart.html#ce-quickstart](https://docs.confluent.io/platform/current/quickstart/ce-quickstart.html#ce-quickstart)
3. An instance of MySQL (this example uses non-secure mysql server running locally)
4. For the delete operation, the command line utility "kafkacat" is used (supports sending a null value in the payload). [This document](https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html) will help with an overview of kafkacat and provides a link to download it. 

### Create MySQL resources 

Create database in MySQL

    CREATE DATABASE `test` /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci */ /*!80016 DEFAULT ENCRYPTION='N' */;

Create MySQL table inside the test2 database:

    CREATE TABLE `orderlines` (
    `id` VARCHAR(40) NOT NULL,
    `product` TEXT,
    `quantity` INT,
    `price` FLOAT,
    `class` TEXT,
    `type` TEXT,
    `newprice` FLOAT,
    PRIMARY KEY(`id`));

### Deploy the JDBC Sink Connector

 1. Visit [https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc)
 2. As described in the above link, run `confluent-hub` to install the connector plugin
 3. At that point, you can install the connector itself within Control Center (described below)

### Produce message to Kafka
Open a console and send a message with a key (required for upsert and delete operations), this will create a topic and schema:

    kafka-avro-console-producer \
    --broker-list localhost:9092 --topic orderlines \
    --property parse.key=true \
    --property key.separator=";" \
    --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"string"},{"name":"product", "type": "string"}, {"name":"quantity","type": "int"}, {"name":"price","type": "float"}, {"name":"class","type": "string"}, {"name":"type","type": "string"}, {"name":"newprice","type": "float"}]}' \
    --property key.schema='{"type":"string"}'

Paste the following string into the console that is waiting for input:

    "abc";{"id": "abc", "product": "bar", "quantity": 14, "price": 14, "class": "superbduh", "type": "sometypytype", "newprice":55.43}

Verify in Confluent Control Center ([http://localhost:9021](http://localhost:9021)) that the topic 'orderlines' has been auto-created with the above statement

### Load Connector 
In Control Center, 

1. Go to Connect on the left side navigation
2. Click on connect-default
3. Click the button for "**Upload Connector Config File**"
4. Upload connector file, [jdbc-sink.json](https://github.com/derplarsen/jdbcsinktomysql/blob/main/jdbc-sink.json)

### Back to data loading / verification

Check to see if the record was loaded into the database / table:

     select * from orderlines;

Put in a new record with the same key but different values (upsert)

    "abc";{"id": "abc", "product": "bar", "quantity": 12, "price": 15, "class": "superbclass", "type": "some type", "newprice":55.44}

Verify that it came through properly:

    select * from orderlines;

Add another record:

    "def";{"id": "def", "product": "bar", "quantity": 12, "price": 25, "class": "ummm", "type": "sometype", "newprice":54.43}
Verify that it came through properly:

    select * from orderlines;

Delete record "abc" by sending in a tombstone record:

    echo "abc;" | kafkacat -b localhost:9092 -t orders -P -Z -K;

See that it got deleted:

    select * from orderlines;

## Additional Notes:

 - To do the update and delete operations, the *key must be specified in
   the payload.*  
 - To automatically create the table or alter the table,
   the avro schema specified must provide a default value, or some way
   to specify how to handle null values.  
- If you're having issues,  run `confluent local services connect log` to easily view the log for the connector.
- When doing this, my MySQL Workbench application did not like showing the value
   for "id" in my records. I was able to view it with the mysql cli
   without a problem. I'm still unsure why.
- Robin Moffatt has put together a very comprehensive example here https://rmoff.net/2021/03/12/kafka-connect-jdbc-sink-deep-dive-working-with-primary-keys/ 
