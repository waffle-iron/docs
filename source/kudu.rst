Kafka Connect Kudu
===================

A Connector and Sink to write events from Kafka to kudu.

The connector takes the value from the Kafka Connect SinkRecords and inserts a new entry to Kudu.

Prerequisites
-------------

- Confluent 2.0
- Kudu 0.7
- Java 1.8
- Scala 2.11

Setup
-----

Kudu Setup
~~~~~~~~~~~

Download and check Kudu QuickStart VM starts up.

.. sourcecode:: bash

    curl -s https://raw.githubusercontent.com/cloudera/kudu-examples/master/demo-vm-setup/bootstrap.sh | bash

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

In ``/etc/kafka/server.properties`` add the following so we can delete topics.

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

The prebuilt jars can be taken from here and `here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
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

Sink Connector QuickStart
-------------------------

Kudu Table
~~~~~~~~~~

The sink currently expects precreated tables in Kudu.


.. sourcecode:: bash

    #demo/demo
    ssh demo@quickstart -t impala-shell

    CREATE TABLE default.kudu_test (id INT,random_field STRING  )
    > TBLPROPERTIES ('kudu.master_addresses'='127.0.0.1', 'kudu.key_columns'='id',
    > 'kudu.table_name'='kudu_test', 'transient_lastDdlTime'='1456744118',
    > 'storage_handler'='com.cloudera.kudu.hive.KuduStorageHandler')
    exit;

.. note:: The sink will fail to start if the tables matching the topics do not already exist.

Sink Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Next we start the connector in standalone mode. This useful for testing
and one of jobs, usually you'd run in distributed mode to get fault
tolerance and better performance.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Since we are in standalone mode we'll create a file called ``kudu-sink.properties`` with the contents below:

.. sourcecode:: bash

    name=kudu-sink
    connector.class=com.datamountaineer.streamreactor.connect.kudu.KuduSinkConnector
    tasks.max=1
    connect.kudu.master=quickstart
    topics=kudu_test

This configuration defines:

1.  The name of the sink.
2.  The sink class.
3.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in
    the source topics otherwise tasks will be idle.
4.  The source kafka topics to take events from.


Starting the Sink Connector (Standalone)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now we are ready to start the Kudu sink Connector in standalone mode.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-Kudu-0.1-all.jar
    #Start the connector in standalone mode, passing in two properties files, the first for the schema registry, kafka
    #and zookeeper and the second with the connector properties.
    ➜  bin/connect-standalone etc/schema-registry/connect-avro-standalone.properties kudu-sink.properties

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    ➜ java -jar build/libs/kafka-connect-cli-0.2-all.jar get kudu-sink

    #Connector name=kudu-sink
    connector.class=com.datamountaineer.streamreactor.connect.kudu.KuduSinkConnector
    tasks.max=1
    connect.kudu.master=quickstart
    topics=kudu_test
    #task ids: 0

.. sourcecode:: bash

    [2016-05-08 22:00:20,823] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      /  / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
           __ __          __      _____ _       __
          / //_/_  ______/ /_  __/ ___/(_)___  / /__
         / ,< / / / / __  / / / /\__ \/ / __ \/ //_/
        / /| / /_/ / /_/ / /_/ /___/ / / / / / ,<
       /_/ |_\__,_/\__,_/\__,_//____/_/_/ /_/_/|_|


    by Andrew Stevenson
           (com.datamountaineer.streamreactor.connect.kudu.KuduSinkTask:37)
    [2016-05-08 22:00:20,823] INFO KuduSinkConfig values:
        connect.kudu.master = quickstart
     (com.datamountaineer.streamreactor.connect.kudu.KuduSinkConfig:165)
    [2016-05-08 22:00:20,824] INFO Connecting to Kudu Master at quickstart (com.datamountaineer.streamreactor.connect.kudu.KuduWriter$:33)
    [2016-05-08 22:00:20,875] INFO Initialising Kudu writer (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:40)
    [2016-05-08 22:00:20,892] INFO Assigned topics  (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:42)
    [2016-05-08 22:00:20,904] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@b60ba7b finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)

Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``id`` field of type int
and a ``random_field`` of type string.

.. sourcecode:: bash

    bin/kafka-avro-console-producer \
    > --broker-list localhost:9092 --topic kudu_test \
    > --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"},
    {"name":"random_field", "type": "string"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"id": 999, "random_field": "foo"}
    {"id": 888, "random_field": "bar"}

Check for records in Kudu
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    [2016-05-08 22:09:22,065] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
           __ __          __      _____ _       __
          / //_/_  ______/ /_  __/ ___/(_)___  / /__
         / ,< / / / / __  / / / /\__ \/ / __ \/ //_/
        / /| / /_/ / /_/ / /_/ /___/ / / / / / ,<
       /_/ |_\__,_/\__,_/\__,_//____/_/_/ /_/_/|_|


    by Andrew Stevenson
           (com.datamountaineer.streamreactor.connect.kudu.KuduSinkTask:37)
    [2016-05-08 22:09:22,065] INFO KuduSinkConfig values:
        connect.kudu.master = quickstart
     (com.datamountaineer.streamreactor.connect.kudu.KuduSinkConfig:165)
    [2016-05-08 22:09:22,066] INFO Connecting to Kudu Master at quickstart (com.datamountaineer.streamreactor.connect.kudu.KuduWriter$:33)
    [2016-05-08 22:09:22,116] INFO Initialising Kudu writer (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:40)
    [2016-05-08 22:09:22,134] INFO Assigned topics kudu_test (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:42)
    [2016-05-08 22:09:22,148] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@68496440 finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)
    [2016-05-08 22:09:22,276] WARN Slow DNS lookup!  Resolved IP of `quickstart' to 192.168.56.101 in 6704556ns (org.kududb.client.AsyncKuduClient:1711)
    [2016-05-08 22:09:22,432] INFO Discovered tablet Kudu Master for table Kudu Master with partition ["", "") (org.kududb.client.AsyncKuduClient:1230)
    [2016-05-08 22:09:22,476] INFO Written 2 for kudu_test (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:90)
    [2016-05-08 22:09:22,476] INFO Discovered tablet 8340243e03ea4381b680d497be9a6c5e for table kudu_test with partition ["", "") (org.kududb.client.AsyncKuduClient:1230)
    [2016-05-08 22:09:23,555] WARN Slow DNS lookup!  Resolved IP of `quickstart.cloudera' to 192.168.56.101 in 1078859124ns (org.kududb.client.AsyncKuduClient:1711)

In Kudu:

.. sourcecode:: bash

    #demo/demo
    ssh demo@quickstart -t impala-shell

    SELECT * FROM kudu_test;

    Query: select * FROM kudu_test
    +-----+--------------+
    | id  | random_field |
    +-----+--------------+
    | 888 | bar          |
    | 999 | foo          |
    +-----+--------------+
    Fetched 2 row(s) in 0.14s

Now stop the connector.

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Connectors can be deployed distributed mode. In this mode one or many connectors are started on the same or different
hosts with the same cluster id. The cluster id can be found in ``etc/schema-registry/connect-avro-distributed.properties.``

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

For this quick-start we will just use one host.

Now start the connector in distributed mode, this time we only give it one properties file for the kafka, zookeeper and
schema registry configurations.

.. sourcecode:: bash

    ➜  confluent-2.0.1/bin/connect-distributed confluent-2.0.1/etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.2-all.jar create Kudu-sink < kudu-sink.properties

If you switch back to the terminal you started the Connector in you should see the Kudu sink being accepted and the task
starting.

Insert the records as before to have them written to Kudu.

Features
--------

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The JDBC sink supports the following:

.. sourcecode:: bash

    <write mode> INTO <target table> SELECT <fields> FROM <source topic> <AUTOCREATE> <PK> <PK_FIELDS> <AUTOEVOLVE>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB

    #Insert mode, select all fields from topicC, auto create tableC and auto evolve, default pks will be created
    INSERT INTO tableC SELECT * FROM topicC AUTOCREATE AUTOEVOLVE

    #Upsert mode, select all fields from topicC, auto create tableC and auto evolve, use field1 and field2 as the primary keys
    UPSERT INTO tableC SELECT * FROM topicC AUTOCREATE PK field1, field2 AUTOEVOLVE


Auto conversion of Connect records to Kudu
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The sink automatically converts incoming Connect records to Kudu inserts.

Topic Routing
~~~~~~~~~~~~~

The sink supports topic routing that allows mapping the messages from topics to a specific table. For example, map a
topic called "bloomberg_prices" to a table called "prices". This mapping is set in the ``connect.kudu.export.route.query``
option.

Example:

.. sourcecode:: sql

    //Select all
    INSERT INTO table1 SELECT * FROM topic1; INSERT INTO tableA SELECT * FROM topicC

.. tip::

    Explicit mapping of topics to tables is required. If not present the sink will not start and fail validation checks.
    Use AUTOCREATE to have the sink create tables for you based on the topic schema.

Field Selection
~~~~~~~~~~~~~~~

The JDBC sink supports field selection and mapping. This mapping is set in the ``connect.kudu.export.route.query`` option.


Examples:

.. sourcecode:: sql

    //Rename or map columns
    INSERT INTO table1 SELECT lst_price AS price, qty AS quantity FROM topicA

    //Select all
    INSERT INTO table1 SELECT * FROM topic1

.. tip:: Check you mappings to ensure the target columns exist.


.. warning::

    Field selection disables evolving the target table if the upstream schema in the Kafka topic changes. By specifying
    field mappings it is assumed the user is not interested in new upstream fields. For example they may be tapping into a
    pipeline for a Kafka stream job and not be intended as the final recipient of the stream.

    If you chose field selection you must include the primary key fields otherwise the insert will fail.

Auto Create Tables
~~~~~~~~~~~~~~~~~~

The sink supports auto creation of tables for each topic.

Any table auto created will have primary keys added. These can either be user specified fields from the topic schema or 3 default
columns set by the sink. If the defaults are requested the sink creates 3 columns, **__connect_topic**,
**__connect_partition** and **__connect_offset**. These columns are set as primary keys. They are filled with the
topic name, partition and offset of the record they came from.

This mapping is set in the ``connect.kudu.export.route.query`` option.

Examples

.. sourcecode:: sql

    //AutoCreate the target table
    INSERT INTO table SELECT * FROM topic AUTOCREATE

    //AuoCreate the target table with USER defined PKS from the record
    INSERT INTO table SELECT * FROM topic AUTOCREATE PK field1, field2

..	note::

    The fields specified as the primary keys must be in the SELECT clause or all fields (*) must be selected

The sink will try and create the table at start up if a schema for the topic is found in the Schema Registry. If no
schema is found the table is created when the first record is received for the topic.

.. tip::

    Pre-create you topics with more than 1 partition so catch as DDL errors such as permission issues beforehand.


Data Type Mappings
~~~~~~~~~~~~~~~~~~

+------------------+------------------+
| Connect Type     | Kudu Data Type   |
+==================+==================+
| INT8             | INT8             |
+------------------+------------------+
| INT16            | INT16            |
+------------------+------------------+
| INT32            | INT32            |
+------------------+------------------+
| INT64            | INT64            |
+------------------+------------------+
| BOOLEAN          | BOOLEAN          |
+------------------+------------------+
| FLOAT32          | FLOAT            |
+------------------+------------------+
| FLOAT64          | FLOAT            |
+------------------+------------------+
| BYTES            | BINARY           |
+------------------+------------------+

Configurations
--------------

``connect.kudu.master``

Specifies a Kudu server.

* Data type : string
* Optional  : no

``connect.kudu.export.route.query``

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

``connect.kudu.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.kudu.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: high
* Default: 10


``connect.kudu.sink.retry.interval``

The interval, in milliseconds between retries if the sink is using ``connect.kudu.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Default : 60000 (1 minute)

``connect.kudu.sink.schema.registry.url``

The url for the Schema registry. This is used to retrieve the latest schema for table creation.

* Type : string
* Importance : high
* Default : http://localhost:8081

``connect.kudu.sink.batch.size``

Specifies how many records to insert together at one time. If the connect framework provides less records when it is
calling the sink it won't wait to fulfill this value but rather execute it.

* Type : int
* Importance : medium
* Defaults : 3000

Example
~~~~~~~

.. sourcecode:: bash

    name=kudu-sink
    connector.class=com.datamountaineer.streamreactor.connect.kudu.KuduSinkConnector
    tasks.max=1
    connect.kudu.master=quickstart
    connect.kudu.export.route.query=INSERT INTO kudu_test SELECT * FROM kudu_test AUTOCREATE PK id
    topics=kudu_test
    connect.kudu.sink.schema.registry.url=http://myhost:8081


Schema Evolution
----------------

TODO

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
