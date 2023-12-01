## Reading from and Writing to External Systems

### Application Consistency Guarantees

- The source and sink connectors of an application need to be integrated with Flink's checkpoints and recovery mechanism and provide certain properties to be able to give meaningful guarantees
- Source connector of the application needs to be able to set its read positions to a previously checkpointed position
- Resettable source connectors guarantees that an application will not lose any data, but are not sufficient to provide end-to-end exactly-once guaranteess
- An application that aims to provide end-to-end exactly-once guarantees requires special sink connectors
- 2 techniques that sink connectors can apply in different situations to achieve exactly-once
  - Idempotent Writes
  - Transactional Writes

#### Idempotent Writes

- An idempotent operation can be performed several times but will only result in a single change
- Idempotent write operations are interesting for streaming applications because they can be performed multiple times without changing the results

#### Transactional Writes

- The idea here is to only write those results to an external sink system that have been computed before the last successful checkpoint
- By only writing data once a checkpoint is completed, the transactional approach does not suffer from the replay inconsistency of the idempotent writes
- It adds latency because results only become visible when a checkpoint completes
- Flink provides 2 building blocks to implement transactional sink connectors
  - Generic **write-ahead-log(WAL)** sink
    - Writes all result records into application state and emits them to the sink system once it receives the notification that a checkpoint was completed
  - **two-phase-commit(2PC)** sink
    - Requires a sink system that offers transactional support or exposes building blocks to emulate transactions
    - For each checkpoint, the sink starts a transaction and appends all received records to the transaction, writing them to the sink system without committing them
    - When it receives the notification that a checkpoint completed, it commits the transaction and materializes the written results


|          | Nonresettable source | Resettable source |
| -------- | -------------------- | ----------------- |
| Any sink | at-most-once         | at-least-once     |
| Idempotent sink | at-most-once  | exactly-once (temporary inconsistencies during recovery) |
| WAL sink | at-most-once         | at-least-once     |
| 2PC sink | at-most-once         | Exactly-once     |

### Provided Connectors

#### Apache Kafka Source Connector

- Kafka organizes event streams as so-called topics
- A topic is an event log that guarantees that events are read in the same order in which they were written
- Flink provides source connectors for all common Kafka versions
- eg. Creating a Flink Kafka source:
```scala
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
properties.setProperty("group.id", "test")
val stream: DataStream[String] = env.addSource(
  new FlinkKafkaConsumer[String](
    "topic",                                // kafka topic to read from, can be single or a list of topics
    new SimpleStringSchema(),               // deserialization strategy, Flink provide Apache Avro and text-based JSON encodings
    properties))                            // configures the Kafka client
```

#### Apache Kafka Sink Connector

- Flink provides sink connectors for all Kafka versions since 0.8
- Flink provides a universal Kafka connector that works for all Kafka versions since 0.11
- eg. Creating a Flink Kafka sink
```scala
val stream: DataStream[String] = ...
val myProducer = new FlinkKafkaProducer[String](
  "localhost:9092",                         // comma-separated string of Kafka broker addresses
  "topic",                                  // the name of the topic to which the data is written
  new SimpleStringSchema)                   // serialization schema: converts the input types of the sink into a byte array
stream.addSink(myProducer)
```

##### At Least Once Guarantees for The Kafka Sink

- The Kafka sink provides at-least-once guarantees under the following conditions
  - Flink’s checkpointing is enabled and all sources of the application are resettable
  - The sink connector throws an exception if a write does not succeed, causing the application to fail and recover (Can change the configuration to retry)
  - The sink connector waits for Kafka to acknowledge in-flight records before completing its checkpoint

##### Exactly Once Guarantees for The Kafka Sink

- Kafka 0.11 introduced support for transactional writes. Due to this feature, Flink’s Kafka sink is also able to provide exactly-once output guarantees given that the sink and Kafka are properly configured
