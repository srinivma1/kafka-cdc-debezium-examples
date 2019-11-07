# Debezium Unwrap SMT Demo

This setup is going to demonstrate how to receive events from SQLServer database and stream them down to a PostgreSQL database and/or an Elasticsearch server using the [Debezium Event Flattening SMT](http://debezium.io/docs/configuration/event-flattening/).


## JDBC Sink

### Topology

```
                   +-------------+
                   |             |
                   | Oracle      |
                   |             |
                   +------+------+
                          |
                          |
                          |
          +---------------v------------------+
          |                                  |
          |           Kafka Connect          |
          |  (Debezium, JDBC connectors)     |
          |                                  |
          +---------------+------------------+
                          |
                          |
                          |
                          |
                  +-------v--------+
                  |                |
                  |   PostgreSQL   |
                  |                |
                  +----------------+


```
We are using Docker Compose to deploy following components
 * Oracle
 * Kafka
 * ZooKeeper
 * Kafka Broker
 * Kafka Connect with [Debezium](http://debezium.io/) and  [JDBC](https://github.com/confluentinc/kafka-connect-jdbc) Connectors. Debezium on the source side and JDBC on the sink side
* PostgreSQL
This assumes Oracle is running on localhost
(or is reachable there, e.g. by means of running it within a VM or Docker container with appropriate port configurations)
and set up with the configuration, users and grants described in the Debezium [Vagrant set-up](https://github.com/srinivma1/oracle-vagrant-box).
Note that the connector is using the XStream API, which requires a license for the Golden Gate product
(which itself is not required be installed, though).

Also you must download the [Oracle instant client for Linux](http://www.oracle.com/technetwork/topics/linuxx86-64soft-092277.html)

Please ensure to download **instantclient-basic-linux.x64-12.2.0.1.0.zip** and **instantclient-sqlplus-linux.x64-12.2.0.1.0.zip** files. Unzip and put it under the directory **_debezium-with-oracle-jdbc/oracle_instantclient_**.

```shell
# Start the topology as defined in http://debezium.io/docs/tutorial/
export DEBEZIUM_VERSION=0.10
docker-compose -f docker-compose-oracle.yaml up --build

# Insert test data
cat debezium-with-oracle-jdbc/init/inventory.sql | docker exec -i dbz_oracle sqlplus debezium/dbz@//localhost:1521/ORCLPDB1
```

Adjust the host name of the database server and the name of the XStream outbound server in `register-oracle.json` as per your environment.
Then register the Debezium Oracle connector and Postgres Connector:

```shell

# Start PostgreSQL connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @jdbc-sink.json

# Start Oracle connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-oracle.json

# Create a test change record
echo "INSERT INTO customers VALUES (NULL, 'John', 'Doe', 'john.doe@example.com');" | docker exec -i dbz_oracle sqlplus debezium/dbz@//localhost:1521/ORCLPDB1

# Consume messages from a Debezium topic
docker-compose -f docker-compose-oracle.yaml exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic CUSTOMERS
    
# Check contents of customers in Oracle

docker exec -i dbz_oracle sqlplus debezium/dbz@//localhost:1521/ORCLPDB1

SQL> select * from customers;

        ID
----------
FIRST_NAME
--------------------------------------------------------------------------------
LAST_NAME
--------------------------------------------------------------------------------
EMAIL
--------------------------------------------------------------------------------
      1066
Anusuya
Mahesh
anu.mahesh@example.com


        ID
----------
FIRST_NAME
--------------------------------------------------------------------------------
LAST_NAME
--------------------------------------------------------------------------------
EMAIL
--------------------------------------------------------------------------------
      1067
Akkur
Subramanian
ar.subramani@example.com


        ID
----------
FIRST_NAME
--------------------------------------------------------------------------------
LAST_NAME
--------------------------------------------------------------------------------
EMAIL
--------------------------------------------------------------------------------
      1001
Sally
Thomas
sally.thomas@acme.com


        ID
----------
FIRST_NAME
--------------------------------------------------------------------------------
LAST_NAME
--------------------------------------------------------------------------------
EMAIL
--------------------------------------------------------------------------------
      1002
George
Bailey
gbailey@foobar.com


        ID
----------
FIRST_NAME
--------------------------------------------------------------------------------
LAST_NAME
--------------------------------------------------------------------------------
EMAIL
--------------------------------------------------------------------------------
      1003
Edward
Walker
ed@walker.com


        ID
----------
FIRST_NAME
--------------------------------------------------------------------------------
LAST_NAME
--------------------------------------------------------------------------------
EMAIL
--------------------------------------------------------------------------------
      1004
Anne
Kretchmar
annek@noanswer.org


        ID
----------
FIRST_NAME
--------------------------------------------------------------------------------
LAST_NAME
--------------------------------------------------------------------------------
EMAIL
--------------------------------------------------------------------------------
      1041
Mahesh
Srinivas
mahesh.srinivas@example.com


        ID
----------
FIRST_NAME
--------------------------------------------------------------------------------
LAST_NAME
--------------------------------------------------------------------------------
EMAIL
--------------------------------------------------------------------------------
      1065
Sowmya
Mahesh
sowmya.m@example.com


        ID
----------
FIRST_NAME
--------------------------------------------------------------------------------
LAST_NAME
--------------------------------------------------------------------------------
EMAIL
--------------------------------------------------------------------------------
      1021
John
Doe
john.doe@example.com


9 rows selected.

# Check contents of customers table in postgres
docker-compose -f docker-compose-oracle.yaml exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB'
WARNING: The DEBEZIUM_VERSION variable is not set. Defaulting to a blank string.
psql (9.6.15)
Type "help" for help.

                      ^
testdb=# \dt
             List of relations
 Schema |   Name    | Type  |    Owner
--------+-----------+-------+--------------
 public | CUSTOMERS | table | postgresuser
(1 row)


testdb=# select * from public."CUSTOMERS";
  ID  |  LAST_NAME  |            EMAIL            | FIRST_NAME
------+-------------+-----------------------------+------------
 1001 | Thomas      | sally.thomas@acme.com       | Sally
 1002 | Bailey      | gbailey@foobar.com          | George
 1003 | Walker      | ed@walker.com               | Edward
 1004 | Kretchmar   | annek@noanswer.org          | Anne
 1041 | Srinivas    | mahesh.srinivas@example.com | Mahesh
 1021 | Doe         | john.doe@example.com        | John
 1065 | Mahesh      | sowmya.m@example.com        | Sowmya
 1066 | Mahesh      | anu.mahesh@example.com      | Anusuya
 1067 | Subramanian | ar.subramani@example.com    | Akkur
(9 rows)

# Modify other records in the database via Oracle client
docker exec -i dbz_oracle sqlplus debezium/dbz@//localhost:1521/ORCLPDB1

# Shut down the cluster
docker-compose -f docker-compose-oracle.yaml down
```
