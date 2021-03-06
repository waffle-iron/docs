.. kafka-connectors:

Kafka Connect Cassandra
=======================

Kafka Connect Cassandra is a Source Connector for reading data from
Cassandra and a Sink Connector for writing data to Cassandra.

Prerequisites
-------------

-  Cassandra 2.2.4
-  Confluent 2.0
-  Java 1.8
-  Scala 2.11

Setup
-----

Before we can do anything, including the QuickStart we need to install
Cassandra and the Confluent platform.

Cassandra Setup
~~~~~~~~~~~~~~~

First download and install Cassandra if you don't have a compatible
cluster available.

.. sourcecode:: bash

    #make a folder for cassandra
    mkdir cassandra

    #Download Cassandra
    wget http://apache.cs.uu.nl/cassandra/3.5/apache-cassandra-3.5-bin.tar.gz

    #extract archive to cassandra folder
    tar -xvf apache-cassandra-3.5-bin.tar.gz -C cassandra

    #Set up environment variables
    export CASSANDRA_HOME=~/cassandra/apache-cassandra-3.5-bin
    export PATH=$PATH:$CASSANDRA_HOME/bin

    #Start Cassandra
    sudo sh ~/cassandra/bin/cassandra

Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    mkdir confluent

    #download confluent
    wget http://packages.confluent.io/archive/2.0/confluent-2.0.1-2.11.7.tar.gz

    #extract archive to confluent folder
    tar -xvf confluent-2.0.1-2.11.7.tar.gz -C confluent

    #setup variables
    export CONFLUENT_HOME=~/confluent/confluent-2.0.1

Enable topic deletion.

In ``/etc/kafka/server.properties`` add the following so we can delete
topics.

.. sourcecode:: bash

    delete.topic.enable=true

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    bin/kafka-server-start etc/kafka/server.properties &
    bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt jars can be taken from here and
`here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
or from `Maven <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    git clone https://github.com/datamountaineer/stream-reactor
    cd stream-reactor
    gradle fatJar

    ##Build the CLI for interacting with Kafka connectors
    git clone https://github.com/datamountaineer/kafka-connect-tools
    cd kafka-connect-tools
    gradle fatJar

Source Connector
----------------

The Cassandra source connector allows you to extract entries from Cassandra with the CQL driver and write them into a
Kafka topic.

Each table specified in the configuration is polled periodically and each record from the result is converted to a Kafka
Connect record. These records are then written to Kafka by the Kafka Connect framework.

The source connector operates in two modes:

1. Bulk - Each table is selected in full each time it is polled.
2. Incremental - Each table is querying with lower and upper bounds to
   extract deltas.

In incremental mode the column used to identify new or delta rows has to be provided. This column must be of CQL Type
Timestamp. Due to Cassandra's and CQL restrictions this should be a primary key or part of a composite primary keys.
ALLOW\_FILTERING can also be supplied as an configuration.

.. note::

    TimeUUIDs are converted to strings. Use the `UUIDs <https://docs.datastax.com/en/drivers/java/2.0/com/datastax/driver/core/utils/UUIDs.html>`__
    helpers to convert to Dates.

Source Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~~~

To see the basic functionality of the Source connector we will start with the Bulk import mode.

Test data
^^^^^^^^^

Once you have installed and started Cassandra create a table to extract records from. This snippet creates a table called
orders and inserts 3 rows representing fictional orders or some options and futures on a trading platform.

Start the Cassandra cql shell

.. sourcecode:: bash

    ➜  bin ./cqlsh
    Connected to Test Cluster at 127.0.0.1:9042.
    [cqlsh 5.0.1 | Cassandra 3.0.2 | CQL spec 3.3.1 | Native protocol v4]
    Use HELP for help.
    cqlsh>

Execute the following:

.. sourcecode:: sql

    CREATE KEYSPACE demo WITH REPLICATION = {'class' : 'SimpleStrategy', 'replication_factor' : 3};
    use demo;

    create table orders (id int, created timeuuid, product text, qty int, price float, PRIMARY KEY (id, created))
    WITH CLUSTERING ORDER BY (created asc);

    INSERT INTO orders (id, created, product, qty, price) VALUES (1, now(), 'OP-DAX-P-20150201-95.7', 100, 94.2);
    INSERT INTO orders (id, created, product, qty, price) VALUES (2, now(), 'OP-DAX-C-20150201-100', 100, 99.5);
    INSERT INTO orders (id, created, product, qty, price) VALUES (3, now(), 'FU-KOSPI-C-20150201-100', 200, 150);

    SELECT * FROM orders;

     id | created                              | price | product                 | qty
    ----+--------------------------------------+-------+-------------------------+-----
      1 | 17fa1050-137e-11e6-ab60-c9fbe0223a8f |  94.2 |  OP-DAX-P-20150201-95.7 | 100
      2 | 17fb6fe0-137e-11e6-ab60-c9fbe0223a8f |  99.5 |   OP-DAX-C-20150201-100 | 100
      3 | 17fbbe00-137e-11e6-ab60-c9fbe0223a8f |   150 | FU-KOSPI-C-20150201-100 | 200

    (3 rows)

    (3 rows)

Source Connector Configuration (Bulk)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Next we start the connector in standalone mode. This useful for testing and one of jobs, usually you'd run in
distributed mode to get fault tolerance and better performance.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stoping, starting and updating the
configuration.

Since we are in standalone mode we'll create a file called ``cassandra-source-bulk-orders.properties`` with the contents below:

.. sourcecode:: bash

    name=cassandra-source-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    cassandra.key.space=demo
    connect.cassandra.import.route.query=INSERT INTO orders-topic SELECT * FROM orders
    cassandra.import.mode=bulk
    cassandra.authentication.mode=username_password
    cassandra.contact.points=localhost
    cassandra.username=cassandra
    cassandra.password=cassandra

This configuration defines:

1. The name of the connector, must be unique.
2. The name of the connector class.
3. The keyspace (demo) we are connecting to.
4. The table to topic import map. This allows you to route tables to different topics. Each mapping is comma separated
   and for each mapping the table and topic are separated by a colon, if no topic is provided the records from the table
   will be routed to a topic matching the table name. In this example the orders table records are routed to the topic
   orders-topic. This property sets the tables to import!
5. The import mode, either incremental or bulk.
6. The authentication mode, this is either none or username\_password. We haven't enabled this on our Cassandra install but you should.
7. The ip or host name of the nodes in the Cassandra cluster to connect to.
8. Username and password, ignored unless you have set Cassandra to use the PasswordAuthenticator.

Starting the Source Connector (Standalone)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now we are ready to start the Cassandra Source Connector in standalone mode.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-cassandra-0.1-all.jar
    #Start the connector in standalone mode, passing in two properties files, the first for the schema registry, kafka
    #and zookeeper and the second with the connector properties.
    ➜  bin/connect-standalone etc/schema-registry/connect-avro-standalone.properties cassandra-source-bulk-orders.properties

We can use the CLI to check if the connector is up but you should be able to see this in logs.

.. sourcecode:: bash

    [2016-05-06 13:52:28,178] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
       ______                                __           _____
      / ____/___ _______________ _____  ____/ /________ _/ ___/____  __  _______________
     / /   / __ `/ ___/ ___/ __ `/ __ \/ __  / ___/ __ `/\__ \/ __ \/ / / / ___/ ___/ _ \
    / /___/ /_/ (__  |__  ) /_/ / / / / /_/ / /  / /_/ /___/ / /_/ / /_/ / /  / /__/  __/
    \____/\__,_/____/____/\__,_/_/ /_/\__,_/_/   \__,_//____/\____/\__,_/_/   \___/\___/

    By Andrew Stevenson. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceTask:64)
    [2016-05-06 13:34:41,193] INFO Attempting to connect to Cassandra cluster at localhost and create keyspace demo. (com.datamountaineer.streamreactor.connect.cassandra.CassandraConnection$:49)
    [2016-05-06 13:34:41,263] INFO Using username_password. (com.datamountaineer.streamreactor.connect.cassandra.CassandraConnection$:83)
    [2016-05-06 13:34:41,459] INFO Did not find Netty's native epoll transport in the classpath, defaulting to NIO. (com.datastax.driver.core.NettyUtil:83)
    [2016-05-06 13:34:41,823] INFO Using data-center name 'datacenter1' for DCAwareRoundRobinPolicy (if this is incorrect, please provide the correct datacenter name with DCAwareRoundRobinPolicy constructor) (com.datastax.driver.core.policies.DCAwareRoundRobinPolicy:95)
    [2016-05-06 13:34:41,824] INFO New Cassandra host localhost/127.0.0.1:9042 added (com.datastax.driver.core.Cluster:1475)
    [2016-05-06 13:34:41,868] INFO Connection to Cassandra established. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceTask:87)
    ....


.. sourcecode:: bash

    ➜ java -jar build/libs/kafka-connect-cli-0.2-all.jar get cassandra-source-orders
    #Connector `cassandra-source-orders`:
    connect.cassandra.key.space=demo
    name=cassandra-source-orders
    connect.cassandra.import.mode=bulk
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    connect.cassandra.authentication.mode=username_password
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra
    connect.cassandra.import.route.query=INSERT INTO orders-topic SELECT * FROM orders
    #task ids: 0




Check for Source Records in Kafka
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    [2016-05-06 13:34:41,923] INFO Source task Thread[WorkerSourceTask-cassandra-source-orders-0,5,main] finished initialization and start (org.apache.kafka.connect.runtime.WorkerSourceTask:342)
    [2016-05-06 13:34:41,927] INFO Query SELECT * FROM demo.orders WHERE created > maxTimeuuid(?) AND created <= minTimeuuid(?)  ALLOW FILTERING executing with bindings (1900-01-01 00:19:32+0019, 2016-05-06 13:34:41+0200). (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:156)
    [2016-05-06 13:34:41,948] INFO Querying returning results for demo.orders. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:185)
    [2016-05-06 13:34:41,958] INFO Found 3. Draining entries to batchSize 100. (com.datamountaineer.streamreactor.connect.queues.QueueHelpers$:45)
    [2016-05-06 13:34:41,958] INFO Processed 3 rows for table orders-topic.orders (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:206)

We can then use the kafka-avro-console-consumer to see what's in the kafka topic we have routed the order table to.

.. sourcecode:: bash

    ➜  confluent-2.0.1/bin/kafka-avro-console-consumer \
    --zookeeper localhost:2181 \
    --topic orders-topic \
    --from-beginning
    {"id":{"int":1},"created":{"string":"17fa1050-137e-11e6-ab60-c9fbe0223a8f"},"price":{"float":94.2},"product":{"string":"OP-DAX-P-20150201-95.7"},"qty":{"int":100}}
    {"id":{"int":2},"created":{"string":"17fb6fe0-137e-11e6-ab60-c9fbe0223a8f"},"price":{"float":99.5},"product":{"string":"OP-DAX-C-20150201-100"},"qty":{"int":100}}
    {"id":{"int":3},"created":{"string":"17fbbe00-137e-11e6-ab60-c9fbe0223a8f"},"price":{"float":150.0},"product":{"string":"FU-KOSPI-C-20150201-100"},"qty":{"int":200}}

3 row as expected.

Now stop the connector.

.. note::

    Next time the Connector polls another 3 would be pulled in. In our example the default poll interval is set to
    1 minute. So in 1 minute we'd get rows again.

.. note:: The created field in a TimeUUID is Cassandra, this represented as a string in the Kafka Connect schema.


Source Connector Configuration (Incremental)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The configuration is similar to before but this time we will perform an incremental load. Below is the configuration.
Create a file called ``cassandra-source-incr-orders.properties`` and add the following content:

.. sourcecode:: bash

    name=cassandra-source-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    connect.cassandra.key.space=demo
    connect.cassandra.import.route.query=INSERT INTO orders-topic SELECT * FROM orders PK created
    connect.cassandra.import.mode=incremental
    connect.cassandra.authentication.mode=username_password
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra

There are two changes from the previous configuration:

1. We have added the timestamp column ``created`` to the ``connect.cassandra.import.route.query``. This identifies the
   column used in the where clause with the lower and upper bounds.
2. The ``connect.cassandra.import.mode`` has been set to ``incremental``.

.. note::

    Only Cassandra columns with data type Timeuuid are supported for incremental mode. The column must also be either
    the primary key or part of the compound key. If it's part of the compound key this will introduce a full scan with
    ALLOW\_FILTERING added to the query.

We can reuse the 3 records inserted into Cassandra earlier but lets clean out the target Kafka topic.

.. note::

    You must delete.topics.enable in etc/kafka/server.properties and shutdown any consumers of this topic for this to
    take effect.

.. sourcecode:: bash

    #Delete the topic
    ➜  confluent-2.0.1/bin/kafka-topics --zookeeper localhost:2181 --topic orders-topic --delete

Starting the Connector (Distributed)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Connectors can be deployed distributed mode. In this mode one or many connectors are started on the same or different
hosts with the same cluster id. The cluster id can be found in ``etc/schema-registry/connect-avro-distributed.properties``.

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

For this quick-start we will just use one host.

Now start the connector in distributed mode, this time we only give it one properties file for the kafka, zookeeper
and schema registry configurations.

.. sourcecode:: bash

    ➜  confluent-2.0.1/bin/connect-distributed confluent-2.0.1/etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to post in our incremental properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.2-all.jar create cassandra-source-orders < cassandra-source-incr-orders.properties

    #Connector `cassandra-source-orders`:
    connect.cassandra.key.space=demo
    name=cassandra-source-orders
    connect.cassandra.import.mode=incremental
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    connect.cassandra.authentication.mode=username_password
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra
    connect.cassandra.import.route.query=INSERT INTO orders-topic SELECT * FROM orders PK created
    #task ids: 0

If you switch back to the terminal you started the Connector in you should see the Cassandra Source being accepted and
the task starting and processing the 3 existing rows.

.. sourcecode:: bash

    [2016-05-06 13:44:33,132] INFO Source task Thread[WorkerSourceTask-cassandra-source-orders-0,5,main] finished initialization and start (org.apache.kafka.connect.runtime.WorkerSourceTask:342)
    [2016-05-06 13:44:33,137] INFO Query SELECT * FROM demo.orders WHERE created > maxTimeuuid(?) AND created <= minTimeuuid(?)  ALLOW FILTERING executing with bindings (2016-05-06 09:23:28+0200, 2016-05-06 13:44:33+0200). (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:156)
    [2016-05-06 13:44:33,151] INFO Querying returning results for demo.orders. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:185)
    [2016-05-06 13:44:33,160] INFO Processed 3 rows for table orders-topic.orders (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:206)
    [2016-05-06 13:44:33,160] INFO Found 3. Draining entries to batchSize 100. (com.datamountaineer.streamreactor.connect.queues.QueueHelpers$:45)
    [2016-05-06 13:44:33,197] WARN Error while fetching metadata with correlation id 0 : {orders-topic=LEADER_NOT_AVAILABLE} (org.apache.kafka.clients.NetworkClient:582)
    [2016-05-06 13:44:33,406] INFO Found 0. Draining entries to batchSize 100. (com.datamountaineer.streamreactor.connect.queues.QueueHelpers$:45)

Check Kafka, 3 rows as before.

.. sourcecode:: bash

    ➜  confluent-2.0.1/bin/kafka-avro-console-consumer \
    --zookeeper localhost:2181 \
    --topic orders-topic \
    --from-beginning
    {"id":{"int":1},"created":{"string":"Thu May 05 13:24:22 CEST 2016"},"price":{"float":94.2},"product":{"string":"DAX-P-20150201-95.7"},"qty":{"int":100}}
    {"id":{"int":2},"created":{"string":"Thu May 05 13:26:21 CEST 2016"},"price":{"float":99.5},"product":{"string":"OP-DAX-C-20150201-100"},"qty":{"int":100}}
    {"id":{"int":3},"created":{"string":"Thu May 05 13:26:44 CEST 2016"},"price":{"float":150.0},"product":{"string":"FU-KOSPI-C-20150201-100"},"qty":{"int":200}}

The source tasks will continue to poll but not pick up any new rows yet.

.. code-block::bash

    INFO Query SELECT * FROM demo.orders WHERE created > ? AND created <= ?  ALLOW FILTERING executing with bindings (Thu May 05 13:26:44 CEST 2016, Thu May 05 21:19:38 CEST 2016). (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:152)
    INFO Querying returning results for demo.orders. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:181)
    INFO Processed 0 rows for table orders-topic.orders (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:202)

Inserting new data
''''''''''''''''''

Now lets insert a row into the Cassandra table. Start the CQL shell.

.. code-block:: bash

    ➜  bin ./cqlsh
    Connected to Test Cluster at 127.0.0.1:9042.
    [cqlsh 5.0.1 | Cassandra 3.0.2 | CQL spec 3.3.1 | Native protocol v4]
    Use HELP for help.

Execute the following:

.. code-block:: sql

    use demo;

    INSERT INTO orders (id, created, product, qty, price) VALUES (4, now(), 'FU-DATAMOUNTAINEER-C-20150201-100', 500, 10000);

    SELECT * FROM orders;

     id | created                              | price | product                           | qty
    ----+--------------------------------------+-------+-----------------------------------+-----
      1 | 17fa1050-137e-11e6-ab60-c9fbe0223a8f |  94.2 |            OP-DAX-P-20150201-95.7 | 100
      2 | 17fb6fe0-137e-11e6-ab60-c9fbe0223a8f |  99.5 |             OP-DAX-C-20150201-100 | 100
      4 | 02acf5d0-1380-11e6-ab60-c9fbe0223a8f | 10000 | FU-DATAMOUNTAINEER-C-20150201-100 | 500
      3 | 17fbbe00-137e-11e6-ab60-c9fbe0223a8f |   150 |           FU-KOSPI-C-20150201-100 | 200

    (4 rows)
    cqlsh:demo>

Check the logs.

.. sourcecode:: bash

    [2016-05-06 13:45:33,134] INFO Query SELECT * FROM demo.orders WHERE created > maxTimeuuid(?) AND created <= minTimeuuid(?)  ALLOW FILTERING executing with bindings (2016-05-06 13:31:37+0200, 2016-05-06 13:45:33+0200). (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:156)
    [2016-05-06 13:45:33,137] INFO Querying returning results for demo.orders. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:185)
    [2016-05-06 13:45:33,138] INFO Processed 1 rows for table orders-topic.orders (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:206)
    [2016-05-06 13:45:33,138] INFO Found 0. Draining entries to batchSize 100. (com.datamountaineer.streamreactor.connect.queues.QueueHelpers$:45)

Check Kafka.

.. sourcecode:: bash

    ➜  confluent confluent-2.0.1/bin/kafka-avro-console-consumer \
    --zookeeper localhost:2181 \
    --topic orders-topic \
    --from-beginning

    {"id":{"int":1},"created":{"string":"17fa1050-137e-11e6-ab60-c9fbe0223a8f"},"price":{"float":94.2},"product":{"string":"OP-DAX-P-20150201-95.7"},"qty":{"int":100}}
    {"id":{"int":2},"created":{"string":"17fb6fe0-137e-11e6-ab60-c9fbe0223a8f"},"price":{"float":99.5},"product":{"string":"OP-DAX-C-20150201-100"},"qty":{"int":100}}
    {"id":{"int":3},"created":{"string":"17fbbe00-137e-11e6-ab60-c9fbe0223a8f"},"price":{"float":150.0},"product":{"string":"FU-KOSPI-C-20150201-100"},"qty":{"int":200}}
    {"id":{"int":4},"created":{"string":"02acf5d0-1380-11e6-ab60-c9fbe0223a8f"},"price":{"float":10000.0},"product":{"string":"FU-DATAMOUNTAINEER-C-20150201-100"},"qty":{"int":500}}

Bingo, we have our extra row.

Sink Connector
--------------

The Cassandra Sink allows you to write events from Kafka to Cassandra.

The connector converts the value from the Kafka Connect SinkRecords to Json and uses Cassandra's JSON insert
functionality to insert the rows.

The task expects pre-created tables in Cassandra. Like the source connector the sink allows mapping of topics to tables.

.. note:: The table and keyspace must be created before hand!
.. note:: If the target table has TimeUUID fields the payload string for the corresponding field in Kafka must be a UUID.


Sink Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~

For the quick-start we will reuse the order-topic we created for the
source.

Sink Connector Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The sink configuration is similar to the source, they share most of the same configuration options. Create a file called
cassandra-sink-distributed-orders.properties with contents below.

.. sourcecode:: bash

    name=cassandra-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.cassandra.export.route.query=INSERT INTO orders_write_back SELECT * FROM orders-topic
    connect.cassandra.contact.points=localhost
    connect.cassandra.port=9042
    connect.cassandra.key.space=demo
    connect.cassandra.authentication.mode=username_password
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra

.. note:: All tables must be in the same keyspace.

.. note::

    If a topic specified in the topics configuration option is not present in the ``connect.cassandra.export.route.query``
    the the topic name will be used.

Cassandra Tables
^^^^^^^^^^^^^^^^

The sink expects the tables it's configured to write to are already present in Cassandra. Lets create our table for the sink.

.. sourcecode:: bash

    use demo;
    create table orders_write_back (id int, created timeuuid, product text, qty int, price float, PRIMARY KEY \
    (id, created)) WITH CLUSTERING ORDER BY (created asc);
    SELECT * FROM orders_write_back;

     id | created | price | product | qty
    ----+---------+-------+---------+-----

    (0 rows)
    cqlsh:demo>

Starting the Sink Connector (Distributed)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Again will start in distributed mode.

.. sourcecode:: bash

    ➜  confluent-2.0.1/bin/connect-distributed etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.3-all.jar create cassandra-sink-orders < cassandra-sink-distributed-orders.properties

    #Connector `cassandra-sink-orders`:
    name=cassandra-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.cassandra.export.route.query=INSERT INTO orders_write_back SELECT * FROM orders-topic
    connect.cassandra.contact.points=localhost
    connect.cassandra.port=9042
    connect.cassandra.key.space=demo
    connect.cassandra.authentication.mode=username_password
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra
    #task ids: 0

Now check the logs to see if we started the sink.

.. sourcecode:: bash

    [2016-05-06 13:52:28,178] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
           ______                                __           _____ _       __
          / ____/___ _______________ _____  ____/ /________ _/ ___/(_)___  / /__
         / /   / __ `/ ___/ ___/ __ `/ __ \/ __  / ___/ __ `/\__ \/ / __ \/ //_/
        / /___/ /_/ (__  |__  ) /_/ / / / / /_/ / /  / /_/ /___/ / / / / / ,<
        \____/\__,_/____/____/\__,_/_/ /_/\__,_/_/   \__,_//____/_/_/ /_/_/|_|

     By Andrew Stevenson. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkTask:50)
    [2016-05-06 13:52:28,179] INFO Attempting to connect to Cassandra cluster at localhost and create keyspace demo. (com.datamountaineer.streamreactor.connect.cassandra.CassandraConnection$:49)
    [2016-05-06 13:52:28,179] INFO Using username_password. (com.datamountaineer.streamreactor.connect.cassandra.CassandraConnection$:83)
    [2016-05-06 13:52:28,187] WARN You listed localhost/0:0:0:0:0:0:0:1:9042 in your contact points, but it wasn't found in the control host's system.peers at startup (com.datastax.driver.core.Cluster:2105)
    [2016-05-06 13:52:28,211] INFO Using data-center name 'datacenter1' for DCAwareRoundRobinPolicy (if this is incorrect, please provide the correct datacenter name with DCAwareRoundRobinPolicy constructor) (com.datastax.driver.core.policies.DCAwareRoundRobinPolicy:95)
    [2016-05-06 13:52:28,211] INFO New Cassandra host localhost/127.0.0.1:9042 added (com.datastax.driver.core.Cluster:1475)
    [2016-05-06 13:52:28,290] INFO Initialising Cassandra writer. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraJsonWriter:40)
    [2016-05-06 13:52:28,295] INFO Preparing statements for orders-topic. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraJsonWriter:62)
    [2016-05-06 13:52:28,305] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@37e65d57 finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)
    [2016-05-06 13:52:28,331] INFO Source task Thread[WorkerSourceTask-cassandra-source-orders-0,5,main] finished initialization and start (org.apache.kafka.connect.runtime.WorkerSourceTask:342)

Now check Cassandra

.. sourcecode:: bash

    use demo;
    SELECT * FROM orders_write_back;

     id | created                              | price | product                           | qty
    ----+--------------------------------------+-------+-----------------------------------+-----
      1 | 17fa1050-137e-11e6-ab60-c9fbe0223a8f |  94.2 |            OP-DAX-P-20150201-95.7 | 100
      2 | 17fb6fe0-137e-11e6-ab60-c9fbe0223a8f |  99.5 |             OP-DAX-C-20150201-100 | 100
      4 | 02acf5d0-1380-11e6-ab60-c9fbe0223a8f | 10000 | FU-DATAMOUNTAINEER-C-20150201-100 | 500
      3 | 17fbbe00-137e-11e6-ab60-c9fbe0223a8f |   150 |           FU-KOSPI-C-20150201-100 | 200

    (4 rows)

Bingo, our 4 rows!

Features
--------

Source Connector
~~~~~~~~~~~~~~~~

The source uses Cassandra's executeAysnc functionality. This is non blocking. For the source,
the when the result returns it is iterated over and rows added to a internal queue. This queue is then drained by the
connector and written to Kafka.

Data Types
^^^^^^^^^^

The source connector supports copying tables in bulk and incrementally to Kafka.

The following CQL data types are supported:

+-------------+---------------------+
| CQL Type    | Connect Data Type   |
+=============+=====================+
| TimeUUID    | Optional String     |
+-------------+---------------------+
| UUID        | Optional String     |
+-------------+---------------------+
| Inet        | Optional String     |
+-------------+---------------------+
| Ascii       | Optional String     |
+-------------+---------------------+
| Text        | Optional String     |
+-------------+---------------------+
| Timestamp   | Optional String     |
+-------------+---------------------+
| Date        | Optional String     |
+-------------+---------------------+
| Tuple       | Optional String     |
+-------------+---------------------+
| UDT         | Optional String     |
+-------------+---------------------+
| Boolean     | Optional Boolean    |
+-------------+---------------------+
| TinyInt     | Optional Int8       |
+-------------+---------------------+
| SmallInt    | Optional Int16      |
+-------------+---------------------+
| Int         | Optional Int32      |
+-------------+---------------------+
| Decimal     | Optional String     |
+-------------+---------------------+
| Float       | Optional Float32    |
+-------------+---------------------+
| Counter     | Optional Int64      |
+-------------+---------------------+
| BigInt      | Optional Int64      |
+-------------+---------------------+
| VarInt      | Optional Int64      |
+-------------+---------------------+
| Double      | Optional Int64      |
+-------------+---------------------+
| Time        | Optional Int64      |
+-------------+---------------------+
| Blob        | Optional Bytes      |
+-------------+---------------------+
| Map         | Optional String     |
+-------------+---------------------+
| List        | Optional String     |
+-------------+---------------------+
| Set         | Optional String     |
+-------------+---------------------+

.. note:: For Map, List and Set the value is extracted from the Cassandra Row and inserted as a JSON string representation.

Modes
^^^^^

The source connector runs in both bulk and incremental mode.

Each mode has a polling interval. This interval determines how often the readers execute queries against the Cassandra
tables. It applies to both incremental and bulk modes. The ``cassandra.import.mode`` setting controls the import behaviour.

Incremental
'''''''''''

In ``incremental`` mode the connector supports querying based on a column in the tables with CQL data type of TimeUUID.

Kafka Connect tracks the latest record it retrieved from each table, so it can start at the correct location on the next
iteration (or in case of a crash). In this case the maximum value of the records returned by the result-set is tracked
and stored in Kafka by the framework. If no offset is found for the table at startup a default timestamp of 1900-01-01
is used. This is then passed to a prepared statement containing a range query. For example:

.. sourcecode:: sql

    SELECT * FROM demo.orders WHERE created > maxTimeuuid(?) AND created <= minTimeuuid(?)

.. warning::::

    If the column used for tracking timestamps is a compound key, ALLOW FILTERING is appended to the query.
    This can have a detrimental performance impact of Cassandra as it is effectively issuing a full scan.

Bulk
''''

In ``bulk`` mode the connector extracts the full table, no where clause is attached to the query.

.. warning::

    Watch out with the poll interval. After each interval the bulk query will be executed again.

Topic Routing
^^^^^^^^^^^^^

The sink supports topic routing that allows mapping the messages from topics to a specific table. For example map
a topic called "bloomberg_prices" to a table called "prices". This mapping is set in the
``connect.cassandra.import.route.query`` and ``connect.cassandra.export.route.query`` option.

.. tip::

    Explicit mapping of topics to tables is required. If not present the sink will not start and fail validation checks.


Sink Connector
~~~~~~~~~~~~~~

The sink connector uses Cassandra's `JSON <http://www.datastax.com/dev/blog/whats-new-in-cassandra-2-2-json-support>`__
insert functionality.

The SinkRecord from Kafka connect is converted to JSON and feed into the prepared statements for inserting into Cassandra.

See DataStax's `documentation <http://cassandra.apache.org/doc/cql3/CQL-2.2.html#insertJson>`__ for type mapping.

Topic Routing
^^^^^^^^^^^^^

The sink supports topic routing that allows mapping the messages from topics to a specific table. For example map
a topic called "bloomberg_prices" to a table called "prices". This mapping is set in the
``connect.cassandra.export.route.query`` option.


.. tip::

    Explicit mapping of topics to tables is required. If not present the sink will not start and fail validation checks.

Field Selection
^^^^^^^^^^^^^^^

The sink supports selecting fields from the source topic or selecting all fields and mapping of these fields to columns
in the target table. For example, map a field called "qty"  in a topic to a column called "quantity" in the target
table.

All fields can be selected by using "*" in the field part of ``connect.cassandra.import.route.query``.

Leaving the column name empty means trying to map to a column in the target table with the same name as the field in the
source topic.

Configurations
--------------

Configurations common to both sink and source are:

``connect.cassandra.contact.points``

Contact points (hosts) in Cassandra cluster.

* Data type: string
* Optional : no

``connect.cassandra.key.space``

Key space the tables to write belong to.

* Data type: string
* Optional : no

``connect.cassandra.port``

Port for the native Java driver.

* Data type: int
* Optional : yes
* Default : 9042

``connect.cassandra.authentication.mode``

Mode to authenticate with. Either username_password or none.

* Data type: string
* Optional : yes
* Default : none

``connect.cassandra.username``

Username to connect to Cassandra with if ``connect.cassandra.authentication.mode`` is set to *username_password*.

* Data type: string
* Optional : yes

``connect.cassandra.password``

Password to connect to Cassandra with if ``connect.cassandra.authentication.mode`` is set to *username_password*.

* Data type: string
* Optional : yes

``connect.cassandra.ssl.enabled``

Enables SSL communication against SSL enable Cassandra cluster.

* Data type: boolean
* Optional : yes
* Default : false

``connect.cassandra.trust.store.password``

Password for truststore.

* Data type: string
* Optional : yes

``connect.cassandra.key.store.path``

Path to truststore.

* Data type: string
* Optional : yes

``connect.cassandra.key.store.password``

Password for key store.

* Data type: string
* Optional : yes

``connect.cassandra.ssl.client.cert.auth``

Path to keystore.

* Data type: string
* Optional : yes

Source Connector Configurations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configurations options specific to the source connector are:

``connect.cassandra.import.poll.interval``


The polling interval between queries against tables for bulk mode in milliseconds.
Default is 1 minute.

* Data type: int
* Optional : yes
* Default  : 10

.. warning::

    WATCH OUT WITH BULK MODE AS MAY REPEATEDLY PULL IN THE SAME DATE.

``connect.cassandra.import.mode``

Either bulk or incremental.

* Data type : string
* Optional  : no


``connect.cassandra.import.route.query``

Kafka connect query language expression. Allows for expressive table to topic routing, field selection and renaming.
In incremental mode the timestampColumn can be specified by ``PK colName``.

Examples:

.. sourcecode:: sql

    INSERT INTO TOPIC1 SELECT * FROM TOPIC1 PK myTimeUUICol

* Data type : string
* Optional  : no

.. warning::

    The timestamp column must be of CQL Type TimeUUID.

``connect.cassandra.import.fetch.size``

The fetch size for the Cassandra driver to read.

* Data type : int
* Optional  : yes
* Default   : 1000

``connect.cassandra.source.task.buffer.size``

The size of the queue for buffering resultset records before write to Kafka.

* Data type : int
* Optional  : yes
* Default   : 10000


``connect.cassandra.source.task.batch.size``

The number of records the source  task should drain from the reader queue.

* Data type : int
* Optional  : yes
* Default   : 1000


Bulk Example
^^^^^^^^^^^^

.. sourcecode:: bash

    name=cassandra-source-orders-bulk
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    connect.cassandra.key.space=demo
    connect.cassandra.import.route.query=INSERT INTO TABLE_X SELECT * FROM TOPIC_Y
    connect.cassandra.import.mode=bulk
    connect.cassandra.authentication.mode=username_password
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra

Incremental Example
^^^^^^^^^^^^^^^^^^^

.. sourcecode:: bash

    name=cassandra-source-orders-incremental
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    connect.cassandra.key.space=demo
    connect.cassandra.import.route.query=INSERT INTO TABLE_X SELECT * FROM TOPIC_Y PK created
    connect.cassandra.import.mode=incremental
    connect.cassandra.authentication.mode=username_password
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra

Sink Connector Configurations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configurations options specific to the sink connector are:

``connect.cassandra.export.route.query``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT field1, field2, field3 as renamedField FROM TOPIC2


* Data Type: string
* Optional : no

``connect.kudu.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.kudu.sink.max.retries``
option. The ``connect.kudu.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Default: ``throw``

Example
^^^^^^^

.. sourcecode:: bash

    name=cassandra-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.cassandra.export.route.query= INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT field1,
    field2, field3 as renamedField FROM TOPIC2
    connect.cassandra.contact.points=localhost
    connect.cassandra.port=9042
    connect.cassandra.key.space=demo
    connect.cassandra.authentication.mode=username_password
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal or fields,
data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules. More information
can be found `here <http://docs.confluent.io/2.0.1/schema-registry/docs/api.html#compatibility>`_.

For the Sink connector, if columns are add to the target Cassandra table and not present in the source topic they will be
set to null by Cassandras Json insert functionality. Columns which are omitted from the JSON value map are treated as a
null insert (which results in an existing value being deleted, if one is present), if a record with the same key is
inserted again.

For the Source connector, at present no column selection is handled, every column from the table is queried to column
additions and deletions are handled in accordance with the compatibility mode of the Schema Registry.

Future releases will support auto creation of tables and adding columns on changes to the topic schema.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
