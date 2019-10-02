# Kafka Connect on Heroku

This proof of concept is intended to demonstrate the use of Kafka Connect to sync the data from Heroku Postgres to Heroku Kafka and from Heroku Kafka to Amazon Redshift using [Confluent Kafka Connect](https://docs.confluent.io/current/connect/index.html).

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
- connect_connect_demo (`heroku kafka:topics:create connect_connect_demo --partitions 3`)

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

**Kafka Connect Connectors**

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






