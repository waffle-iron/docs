Kafka Connect Query Language
============================

The Kafka Connect Query Language is implemented in antlr4 grammar files.

Why ?
-----

A Kafka Connect KCQL (Kafka Connect Query Languages) makes a lot of sense when you need to define mappings between Kafka topics (with Avro records) and external systems as sinks or sources.

Kafka Connect Query Language
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: sql

    INSERT into TARGET_SQL_TABLE SELECT * FROM SOURCE_TOPIC IGNORE a,b,c
    UPSERT into TARGET_SQL_TABLE SELECT ..           // INSERT & UPSERT allowed. Works out PK from DB

    SELECT field1                                 // Project one avro field named field1
    SELECT field1.subfield1                       // Project one avro field from a complex message
    SELECT field1 AS newName                      // Project and renames a field
    SELECT *                                      // Select everything - perfect for avro evolution
    SELECT *, field1 AS newName                   // Select all & rename a field - excellent for avro evolution
    SELECT * IGNORE badField                      // Select all & ignore a field - excellent for avro evolution

    #Other operators

    AUTOCREATE                                    // AUTOCREATE TABLE
    AUTOCREATE PK field1,field2                   // AUTOCREATE with Primary Keys
    BATCH 5000                                    // SET BATCHING TO 5000 records
    AUTOEVOLVE
    AUTOCREATE AUTOEVOLVE

    #Future options

    NOOP | THROW | RETRY                          // Define the error policy