## Processing Streams in Parallel

### Latency VS Throughput

- Latency and throughput are not independent metrics, they affect each other

### Operations on Data Streams

- Operations can be either **stateless** or **stateful**
  - **Stateless**: Do not maintain any internal state, processing of an event does not depend on events seen in the past
    - Easy to parallelize
    - Can be simply restarted and continue processing from where it left off
  - **Stateful**: May maintain information about the events they have received before, and this info can be used in the processing logic of future events, also can be updated by incoming events
    - More challenging to parallelize
    - Operate in a fault-tolerant manner

#### Data Ingestion and Data Egress

- Data ingestion and data egress operations allow the stream processor to communicate with external systems
- **Data ingestion**: Operation of fetching raw data from external sources and converting it into a format suitable for processing
  - Operators that implement data ingestion logic are called **data sources**
- **Data egress**: Operation of producing output in a form suitable for consumption by external systems
  - Operators that implement data egress logic are called **data sinks**

#### Transformation Operations

- 