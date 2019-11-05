
# Debezium Unwrap SMT Demo

This setup is going to demonstrate how to receive events from SQLServer database and stream them down to a MongoDB database using the [Debezium Event Flattening SMT](http://debezium.io/docs/configuration/event-flattening/).

## Table of Contents

* [JDBC Sink](#jdbc-sink)
  * [Topology](#topology)
  * [Usage](#usage)
    * [New record](#new-record)
    * [Record update](#record-update)
    * [Record delete](#record-delete)


## JDBC Sink

### Topology

```
                   +-------------+
                   |             |
                   | SQL Server  |
                   |             |
                   +------+------+
                          |
                          |
                          |
          +---------------v------------------+
          |                                  |
          |           Kafka Connect          |
          |  (Debezium connectors)     |
          |                                  |
          +---------------+------------------+
                          |
                          |
                          |
                          |
                  +-------v--------+
                  |                |
                  |   MongoDB      |
                  |                |
                  +----------------+


```
We are using Docker Compose to deploy following components
* SQLServer
* Kafka
  * ZooKeeper
  * Kafka Broker
  * Kafka Connect with [Debezium](http://debezium.io/) and  [JDBC](https://github.com/confluentinc/kafka-connect-jdbc) Connectors
* MongoDB

### Usage

How to run:

```shell
# Start the application
export DEBEZIUM_VERSION=0.10
docker-compose -f docker-compose.yaml up

# Initialize database and insert test data
cat inventory.sql | docker exec -i tutorial_sqlserver_1 bash -c '/opt/mssql-tools/bin/sqlcmd -U sa -P $SA_PASSWORD'

# Start PostgreSQL connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @mongodb-sink.json

# Start SQLServer connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @source.json
```

Check contents of the SQLServer database:

```shell
docker-compose -f docker-compose-jdbc.yaml exec sqlserver bash -c '/opt/mssql-tools/bin/sqlcmd -U sa -P $SA_PASSWORD -d testDB'
>1  select * from customers
>2 GO
+------+------------+-----------+-----------------------+
| id   | first_name | last_name | email                 |
+------+------------+-----------+-----------------------+
| 1001 | Sally      | Thomas    | sally.thomas@acme.com |
| 1002 | George     | Bailey    | gbailey@foobar.com    |
| 1003 | Edward     | Walker    | ed@walker.com         |
| 1004 | Anne       | Kretchmar | annek@noanswer.org    |
+------+------------+-----------+-----------------------+
```

Verify that the MongoDB database has the same content:

```shell
docker-compose exec mongodb bash -c 'mongo testDB'
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
testDB  0.000GB
> use testDB
switched to db testDB
> show collections
customers
> db.customers.find().pretty()
{
        "_id" : {
                "id" : 1001
        },
        "first_name" : "Sally",
        "last_name" : "Thomas",
        "email" : "sally.thomas@acme.com"
}
{
        "_id" : {
                "id" : 1002
        },
        "first_name" : "George",
        "last_name" : "Bailey",
        "email" : "gbailey@foobar.com"
}
{
        "_id" : {
                "id" : 1003
        },
        "first_name" : "Edward",
        "last_name" : "Walker",
        "email" : "ed@walker.com"
}
{
        "_id" : {
                "id" : 1004
        },
        "first_name" : "Anne",
        "last_name" : "Kretchmar",
        "email" : "annek@noanswer.org"
}
> exit

```


End application:

```shell
# Shut down the cluster
docker-compose -f docker-compose-jdbc.yaml down
