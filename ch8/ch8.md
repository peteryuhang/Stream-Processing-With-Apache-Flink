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