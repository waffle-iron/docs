Kafka Connect Redis
===================

A Connector and Sink to write events from Kafka to Redis. The connector takes the value from the Kafka Connect
SinkRecords and inserts a new entry to Redis.

Prerequisites
-------------

- Confluent 2.0
- Jedis 2.8.1
- Java 1.8
- Scala 2.11

Setup
-----

Redis Setup
~~~~~~~~~~~

Download and install Redis.

.. sourcecode:: bash

    ➜  wget http://download.redis.io/redis-stable.tar.gz
    ➜  tar xvzf redis-stable.tar.gz
    ➜  cd redis-stable
    ➜  sudo make install


Start Redis

.. sourcecode:: bash

    ➜  bin/redis-server

Check Redis is running:

.. sourcecode:: bash

    ➜  redis-cli ping
        PONG
    ➜  sudo service redis-server status

Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    ➜  mkdir confluent

    #download confluent
    ➜  wget http://packages.confluent.io/archive/2.0/confluent-2.0.1-2.11.7.tar.gz

    #extract archive to confluent folder
    ➜  tar -xvf confluent-2.0.1-2.11.7.tar.gz -C confluent

    #setup variables
    ➜  export CONFLUENT_HOME=~/confluent/confluent-2.0.1

Enable topic deletion.

In ``/etc/kafka/server.properties`` add the following to we can delete
topics.

.. sourcecode:: bash

    delete.topic.enable=true

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    ➜  bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    ➜  bin/kafka-server-start etc/kafka/server.properties &
    ➜  bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt jars can be taken from here and
`here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
or from `Maven <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    ➜  git clone https://github.com/datamountaineer/stream-reactor
    ➜  cd stream-reactor
    ➜  gradle fatJar

    ##Build the CLI for interacting with Kafka connectors
    ➜  git clone https://github.com/datamountaineer/kafka-connect-tools
    ➜  cd kafka-connect-tools
    ➜  gradle fatJar

Sink Connector QuickStart
-------------------------

Sink Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Next we start the connector in standalone mode. This useful for testing and one of jobs, usually you'd run in
distributed mode to get fault tolerance and better performance.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Since we are in standalone mode we'll create a file called ``redis-sink.properties`` with the contents below:

.. sourcecode:: bash

    name=redis-sink
    connect.redis.connection.host=localhost
    connect.redis.connection.port=6379
    connector.class=com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkConnector
    tasks.max=1
    topics=person_redis
    connect.redis.export.route.query=INSERT INTO TABLE1 SELECT * FROM person_redis

This configuration defines:

1.  The name of the sink.
2.  The name of the redis host to connect to.
3.  The redis port to connect to.
4.  The sink class.
5.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in
    the source topics otherwise tasks will be idle.
6.  The source kafka topics to take events from.
7.  The field mappings, topic mappings and fields to use a the row key.

Starting the Sink Connector (Standalone)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now we are ready to start the Redis sink Connector in standalone mode.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-redis-0.1-all.jar
    #Start the connector in standalone mode, passing in two properties files, the first for the schema registry, kafka
    #and zookeeper and the second with the connector properties.
    ➜  bin/connect-standalone etc/schema-registry/connect-avro-standalone.properties redis-sink.properties

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    ➜ java -jar build/libs/kafka-connect-cli-0.2-all.jar get redis-sink

    #Connector name=`redis-sink`
    connect.redis.connection.host=localhost
    connect.redis.connection.port=6379
    connector.class=com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkConnector
    tasks.max=1
    topics=person_redis
    connect.redis.export.route.query=INSERT INTO TABLE1 SELECT * FROM person_redis
    #task ids: 0

.. sourcecode:: bash

    [2016-05-08 22:37:05,616] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
        ____           ___      _____ _       __
       / __ \___  ____/ (_)____/ ___/(_)___  / /__
      / /_/ / _ \/ __  / / ___/\__ \/ / __ \/ //_/
     / _, _/  __/ /_/ / (__  )___/ / / / / / ,<
    /_/ |_|\___/\__,_/_/____//____/_/_/ /_/_/|_|


     (com.datamountaineer.streamreactor.connect.redis.sink.config.RedisSinkConfig:165)
    [2016-05-08 22:37:05,641] INFO Settings:
    RedisSinkSettings(RedisConnectionInfo(localhost,6379,None),RedisKey(FIELDS,WrappedArray(firstName, lastName)),PayloadFields(false,Map(firstName -> firstName, lastName -> lastName, age -> age, salary -> income)))
           (com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkTask:65)
    [2016-05-08 22:37:05,687] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@44b24eaa finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)


Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type
string a ``lastnamme`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic person_redis \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.redis" \
      ,"fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}

Check for records in Redis
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    INFO Received record from topic:person_redis partition:0 and offset:0 (com.datamountaineer.streamreactor.connect.redis.sink.writer.RedisDbWriter:48)
    INFO Empty list of records received. (com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkTask:75)

Check the Redis.

.. sourcecode:: bash

    redis-cli

    127.0.0.1:6379> keys *
    1) "John.Smith"
    2) "11"
    3) "10"
    127.0.0.1:6379>
    127.0.0.1:6379> get "John.Smith"
    "{\"firstName\":\"John\",\"lastName\":\"Smith\",\"age\":30,\"income\":4830.0}"


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

Once the connector has started lets use the kafka-connect-tools cli to
post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.2-all.jar create redis-sink < redis-sink.properties

If you switch back to the terminal you started the Connector in you should see the Redis sink being accepted and the
task starting.


Features
--------

The Redis sink writes records from Kafka to Redis.

The sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to have choose selection of fields
   or all fields written to Redis.
2. Topic to table routing.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset is
   used.

Configurations
--------------

``connect.redis.sink.connection.host``

Specifies the Redis server.

* Data type : string
* Optional  : no

``connect.redis.sink.connection.port``

Specifies the Redis server port number.

* Data type : int
* Optional  : no

``connect.redis.sink.connection.password``

Specifies the authorization password.

* Data type : string
* Optional  : yes

``connect.redis.export.route.query``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. Fields
to be used as the row key can be set by specifing the ``PK``. The below example uses field1 and field2 are the row key.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1, field2


Example
~~~~~~~

.. sourcecode:: bash

    name=redis-sink
    connect.redis.connection.host=localhost
    connect.redis.connection.port=6379
    connector.class=com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkConnector
    tasks.max=1
    topics=person_redis
    connect.redis.export.route.query=INSERT INTO TABLE1 SELECT * FROM person_redis

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/2.0.1/schema-registry/docs/api.html#compatibility>`_.

The Redis sink will automatically write and update the Redis table if new fields are added to the source topic,
if fields are removed the Kafka Connect framework will return the default value for this field, dependent of the
compatibility settings of the Schema registry. This value will be put into the Redis column family cell based on the
``connect.redis.sink.fields`` mappings.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
