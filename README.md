# Kafka Connect on Heroku

This proof of concept is intended to demonstrate the use of Kafka Connect to sync the data from Heroku Postgres to Heroku Kafka and from Heroku Kafka to Amazon Redshift using [Confluent Kafka Connect](https://docs.confluent.io/current/connect/index.html).

## Deploy to Heroku

[![Deploy to Heroku](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy)

If you are using `Deploy to Heroku` button you can skip the Heroku prerequisite steps.

**Note**: `web` process will crash until you create a required Kafka Topics (**connect-status, connect-offsets, connect-configs**).

## Prerequisite

**Heroku**
- Heroku Application (`heroku create -a <APP_NAME>`)
- Heroku Postgres (`heroku addons:create heroku-postgresql:<PLAN_NAME> -a <APP_NAME>`)
- Heroku Kafka (`heroku addons:create heroku-kafka:standard-0 -a <APP_NAME>`)
- Heroku APT Buildpack (`heroku buildpacks:add --index=1 https://github.com/amiel/heroku-buildpack-apt#feature/support-adding-keys -a <APP_NAME>`)
- Heroku JVM Buildpack (`heroku buildpacks:add heroku/jvm -a <APP_NAME>`)
- Heroku Config Var `KEYSTORE_PASSWORD` (`heroku config:set KEYSTORE_PASSWORD="<PASSWORD>" -a <APP_NAME>`)
- Heroku Config Var `TRUSTSTORE_PASSWORD` (`heroku config:set TRUSTSTORE_PASSWORD="<PASSWORD>" -a <APP_NAME>`)

**Amazon Redshift Cluster**

Setup a Amazon Redshift Cluster by following the steps [here](https://docs.aws.amazon.com/redshift/latest/gsg/rs-gsg-launch-sample-cluster.html)

**Kafka Topics**
- connect-status (`heroku kafka:topics:create connect-status --partitions 3`)
- connect-offsets (`heroku kafka:topics:create connect-offsets --partitions 3`)
- connect-configs (`heroku kafka:topics:create connect-configs --partitions 3`)
- connect_demo (`heroku kafka:topics:create connect_connect_demo --partitions 3`)

**Heroku Postgres Table**

```
CREATE TABLE demo
(
    id          SERIAL NOT NULL,
    name        VARCHAR(100) NOT NULL,
    age         INTEGER,
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY(id)
);
```

**Amazon Redshift Table**

```
CREATE TABLE connect_demo
(
    id          INTEGER NOT NULL,
    name        VARCHAR(100) NOT NULL,
    age         INTEGER,
    updated_at  TIMESTAMP NOT NULL
);
```

## Usage

**Create Kafka Connect Connectors**

Source Connector 

```
curl -X POST https://<HEROKU_APP_URL>/connectors -H "Content-Type: application/json" --data @- << EOF
{
    "name": "postgres-source",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
      "tasks.max": "1",      
      "connection.url": "jdbc:postgresql://<Postgres-Host>:5432/<Database-Name>?user=<Database-UserName>&password=<DatabasePassword>",
      "mode": "incrementing",
      "incrementing.column.name": "id",
      "topic.prefix": "connect_"      
    }
}
EOF
```

Sink Connector 

```
curl -X POST https://<HEROKU_APP_URL>/connectors -H "Content-Type: application/json" --data @- << EOF
{
    "name": "redshift-sink",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
      "tasks.max": "1",
      "topics": "connect_demo",
      "connection.url": "jdbc:redshift://<Redshift-Host>:5439/<Redshift-Database>?user=<Redshift-UserName>&password=<Redshift-Password>",
      "auto.create": "false"
    }
}
EOF
```

**Delete Kafka Connect Connectors**

```
curl -X DELETE https://<HEROKU_APP_URL>/connectors/<CONNECTOR_NAME> 
```

**Insert some data into Heroku Postgres**

```
INSERT INTO demo (name, age) VALUES ('John', 26);
INSERT INTO demo (name, age) VALUES ('Jane', 24);
INSERT INTO demo (name, age) VALUES ('Julia', 25);
INSERT INTO demo (name, age) VALUES ('Jamie', 22);
INSERT INTO demo (name, age) VALUES ('Jenny', 27);
```

This will sync the inserted data from Heroku Postgres to Heroku Kafka Topic `connect_demo` and sync from Heroku Kafka to Amazon Redshift Table (here in `connect_demo` table) using Kafka Connect.

**JDBC Connector Info**
- https://www.confluent.io/blog/kafka-connect-deep-dive-jdbc-source-connector
- https://docs.confluent.io/current/connect/kafka-connect-jdbc/source-connector/index.html
- https://docs.confluent.io/current/connect/kafka-connect-jdbc/sink-connector/index.html
- https://docs.confluent.io/current/connect/managing/connectors.html#connect-bundled-connectors
- https://docs.confluent.io/current/connect/kafka-connect-jdbc/source-connector/source_config_options.html
- https://docs.confluent.io/current/connect/kafka-connect-jdbc/sink-connector/sink_config_options.html






