# Example of ScyllaDB and Kafka using Change Data Capture

### Clone the repository
```
git clone https://github.com/Chris-Basson-Grundling/scylladb-kafka-trigger-example.git
```
### Enter into the scylladb-kafka-trigger-example
```
cd scylladb-kafka-trigger-example
```

## ScyllaDB Install And Init Table

### Launch ScyllaDB cluster
```
docker-compose -f docker-compose-scylladb.yml up -d
```
### Wait for cluster to be up and in the normal state (UN)
```
docker exec scylla-node1 nodetool status
```
### Importing the keyspace and data automatically
```
docker exec scylla-node1 cqlsh -f /init-data.txt
```

## Confluent Setup And Connector Configuration

### Launch the Confluent services
``` 
docker-compose -f docker-compose-confluent.yml up -d
```
### Wait for all services to initialise then go to confluent GUI

http://localhost:9021

### Add new connection via GUI

``` 
Connect tab >> connect-default >> Add connector >> ScyllaConnector

Fill the “Hosts” with the IP address of one of the Scylla nodes,
you can see it in the output of the nodetool status command, 
docker exec scylla-node1 nodetool status
with port 9042
eg: 172.22.0.4:9042

The “Namespace” is the Keyspace you created before in ScyllaDB.
eg: ks

Notice that it might take a minute or so for ks.my_table to appear.
```

## Test Kafka Messages

``` 
docker exec -ti scylla-node1 cqlsh

INSERT INTO ks.my_table(pk, ck, v) VALUES (1, 2, 1);
INSERT INTO ks.my_table(pk, ck, v) VALUES (1, 2, 2);
INSERT INTO ks.my_table(pk, ck, v) VALUES (1, 2, 4);
```

## Cleanup

### To remove kill and remove the containers
```
docker-compose -f docker-compose-scylladb.yml kill
docker-compose -f docker-compose-scylladb.yml rm -f
docker-compose -f docker-compose-confluent.yml kill
docker-compose -f docker-compose-confluent.yml rm -f
```

## Create ScyllaDB Keyspace, Table and add data manually
```
docker exec -ti scylla-node1 cqlsh

CREATE KEYSPACE ks WITH replication = {'class': 'NetworkTopologyStrategy', 'replication_factor' : 1};

CREATE TABLE ks.my_table (pk int, ck int, v int, PRIMARY KEY (pk, ck, v)) WITH cdc = {'enabled':true};

INSERT INTO ks.my_table(pk, ck, v) VALUES (1, 2, 3);
INSERT INTO ks.my_table(pk, ck, v) VALUES (2, 3, 4);
INSERT INTO ks.my_table(pk, ck, v) VALUES (3, 4, 5);

exit
```

## Manually setup Confluent platform with ScyllaDB CDC Connector

### Download the Confluent platform docker-compose file
``` 
wget -O docker-compose-confluent.yml https://raw.githubusercontent.com/confluentinc/cp-all-in-one/7.3.0-post/cp-all-in-one/docker-compose.yml
```
### Download the ScyllaDB CDC connector
``` 
wget -O scylla-cdc-plugin.jar https://github.com/scylladb/scylla-cdc-source-connector/releases/download/scylla-cdc-source-connector-1.0.1/scylla-cdc-source-connector-1.0.1-jar-with-dependencies.jar
```
### Update the docker-compose-confluent.yml file
``` 
 image: cnfldemos/cp-server-connect-datagen:0.5.3-7.1.0
     hostname: connect
     container_name: connect
+    volumes:
+      - ./scylla-cdc-plugin.jar:/usr/share/java/kafka/plugins/scylla-cdc-plugin.jar
     depends_on:
       - broker
       - schema-registry
```

## General Issues

### To set the aio-max-nr value, add the following line to the /etc/sysctl.conf file:
```
sudo nano /etc/sysctl.conf
```
```
fs.aio-max-nr = 1048576
```

### To activate the new setting, run the following command:
```
sudo sysctl -p /etc/sysctl.conf
```