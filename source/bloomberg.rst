.. figure:: ../images/bb.jpeg
   :alt: 

Kafka Connect Bloomberg
=======================

A Connector to Subscribe to Bloomberg ticker updates and write the data to Kafka. The connector will create and open a Bloomberg session and will subscribe for data updates given the configuration provided. As data is received from Bloomberg it will be added to a blocking queue. When the kafka connect framework polls the source task for data it will drain this queue.

REST OF DOC IS A WORK IN PROGRESS
