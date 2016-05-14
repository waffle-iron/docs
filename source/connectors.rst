.. image:: https://travis-ci.org/datamountaineer/stream-reactor-gradle.svg?branch=master

Kafka Connect
=============

Kafka Connect is a tool to rapidly stream events in and out of Kafka. It has a narrow focus on data ingress in and egress
out of the central nervous system of modern streaming frameworks. It is not an ETL and this separation of concerns allows
developers to quickly build robust, durable and scalable pipelines in and out of Kafka.

Kafka connect forms an integral component in an ETL pipeline when combined with Kafka and a stream processing framework.

Kafka Connect can run either as a standalone process for testing and one-off jobs, or as a distributed, scalable, fault
tolerant service supporting an entire organisations. This allows it to scale down to development, testing, and small
production deployments with a low barrier to entry and low operational overhead, and to scale up to support a large
organisations data pipeline.

Connectors

.. toctree::
   :maxdepth: 3

   bloomberg
   cassandra
   druid
   elastic
   hbase
   kudu
   redis
