Kafka Connect HBase
===================

A Connector and Sink to write events from Kafka to HBase. The connector takes the value from the Kafka Connect SinkRecords
and inserts a new entry to HBase.

Prerequisites
-------------

- Confluent 2.0
- HBase 1.2.0
- Java 1.8
- Scala 2.11

Setup
-----

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

HBase Setup
~~~~~~~~~~~

Download and extract HBase:

.. sourcecode:: bash

    wget https://www.apache.org/dist/hbase/1.2.1/hbase-1.2.1-bin.tar.gz
    tar -xvf hbase-1.2.1-bin.tar.gz -C hbase


Edit ``conf/hbase-site.xml`` and add the following content:

.. sourcecode:: html

    <?xml version="1.0"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
     <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
      </property>
      <property>
        <name>hbase.rootdir</name>
        <value>file:///tmp/hbase</value>
      </property>
      <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/tmp/zookeeper</value>
      </property>
    </configuration>

The ``hbase.cluster.distributed`` is required since when you start hbase it will try and start it's own Zookeeper, but in
this case we want to use Confluents.

Now start HBase and check the logs to ensure it's up:

.. sourcecode:: bash

    bin/start-hbase.sh

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

Sink Connector QuickStart
-------------------------

HBase Table
~~~~~~~~~~~

The sink expects a precreated table in HBase. In the HBase shell create the test table, go to your HBase install location.

.. sourcecode:: bash

    bin/hbase shell
    hbase(main):001:0> create 'person_hbase',{NAME=>'d', VERSIONS=>1}

    hbase(main):001:0> list
    person
    1 row(s) in 0.9530 seconds

    hbase(main):002:0> describe 'person'
    DESCRIPTION
     'person', {NAME => 'd', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'false', DATA_BLOCK_ENCOD true
     ING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION
     _SCOPE => '0'}
    1 row(s) in 0.0810 seconds


Sink Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Next we start the connector in standalone mode. This useful for testing and one of jobs, usually you'd run in distributed
mode to get fault tolerance and better performance.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Since we are in standalone mode we'll create a file called ``hbase-sink.properties`` with the contents below:

.. sourcecode:: bash

    name=person-hbase-test
    connector.class=com.datamountaineer.streamreactor.connect.hbase.HbaseSinkConnector
    tasks.max=1
    topics=TOPIC1
    connect.hbase.sink.column.family=d
    connect.hbase.export.route.query=INSERT INTO person_hbase SELECT * FROM TOPIC1

This configuration defines:

1.  The name of the sink.
2.  The sink class.
3.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in the source topics
    otherwise tasks will be idle.
4.  The source kafka topics to take events from.
5.  The HBase column family to write to.
6.  The field mappings, topic mappings and fields to use a the row key.

Starting the Sink Connector (Standalone)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now we are ready to start the hbase sink Connector in standalone mode.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-hbase-0.1-all.jar
    #Start the connector in standalone mode, passing in two properties files, the first for the schema registry, kafka
    #and zookeeper and the second with the connector properties.
    ➜  bin/connect-standalone etc/schema-registry/connect-avro-standalone.properties hbase-sink.properties

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    ➜ java -jar build/libs/kafka-connect-cli-0.2-all.jar get hbase-sink

    INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\\_,\\\\\\\__,_/_/  /_/\___\\\\\,\/_/ /_/\\_/\__,_/_/_/ /_/\___/\___/_/
          / / / / __ )____ _________ / ___/(_)___  / /__
         / /_/ / __  / __ `/ ___/ _ \\__ \/ / __ \/ //_/
        / __  / /_/ / /_/ (__  )  __/__/ / / / / / ,<
       /_/ /_/_____/\__,_/____/\___/____/_/_/ /_/_/|_|

    By Stefan Bocutiu (com.datamountaineer.streamreactor.connect.hbase.HbaseSinkTask:44)


Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type string
a ``lastnamme`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic TOPIC1 \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.hbase"
      "fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},
      {"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}
    {"firstName": "Anna", "lastName": "Jones", "age":28, "salary": 5430}

Check for records in HBase
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this

.. sourcecode:: bash

    INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@48ffb4dc finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)
    INFO Writing 2 rows to Hbase... (com.datamountaineer.streamreactor.connect.hbase.writers.HbaseWriter:83)

In HBase:

.. sourcecode:: bash

    hbase(main):004:0* scan 'person_hbase'
    ROW                                                  COLUMN+CELL
     Anna\x0AJones                                       column=d:age, timestamp=1463056888641, value=\x00\x00\x00\x1C
     Anna\x0AJones                                       column=d:firstName, timestamp=1463056888641, value=Anna
     Anna\x0AJones                                       column=d:income, timestamp=1463056888641, value=@\xB56\x00\x00\x00\x00\x00
     Anna\x0AJones                                       column=d:lastName, timestamp=1463056888641, value=Jones
     John\x0ASmith                                       column=d:age, timestamp=1463056693877, value=\x00\x00\x00\x1E
     John\x0ASmith                                       column=d:firstName, timestamp=1463056693877, value=John
     John\x0ASmith                                       column=d:income, timestamp=1463056693877, value=@\xB2\xDE\x00\x00\x00\x00\x00
     John\x0ASmith                                       column=d:lastName, timestamp=1463056693877, value=Smith
    2 row(s) in 0.0260 seconds

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

    ➜  java -jar build/libs/kafka-connect-cli-0.2-all.jar create hbase-sink < hbase-sink.properties

If you switch back to the terminal you started the Connector in you
should see the HBase sink being accepted and the task starting.


Features
--------

The HBase sink writes records from Kafka to HBase.

The sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to have choose selection of fields
   or all fields written to HBase.
2. Topic to table routing.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset is
   used.

Configurations
--------------

``connect.hbase.sink.column.family``

Specifies the table column family to use when inserting the new entry columns.

* Data type : string
* Optional  : no

``connect.hbase.export.route.query``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. Fields
to be used as the row key can be set by specifing the ``PK``. The below example uses field1 and field2 are the row key.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1, field2

Example
~~~~~~~

.. sourcecode:: bash

    connect.hbase.sink.column.family=d
    connect.hbase.export.route.query=INSERT INTO person_hbase SELECT * FROM TOPIC1
    connector.class=com.datamountaineer.streamreactor.connect.hbase.HbaseSinkConnector
    tasks.max=1
    topics=TOPIC1
    name=hbase-test

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/2.0.1/schema-registry/docs/api.html#compatibility>`_.

The HBase sink will automatically write and update the HBase table if new fields are added to the source topic,
if fields are removed the Kafka Connect framework will return the default value for this field, dependent of the
compatibility settings of the Schema registry. This value will be put into the HBase column family cell based on the
``connect.hbase.sink.fields`` mappings.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
