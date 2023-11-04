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

- These operations consume one event after the other and apply some transformation to the event data, producing a new output stream
- The transformation logic can be either integrated in the operator or provided by a user-defined function
- Operators can accept multiple inputs and produce multiple output streams
- They can also modify the structure of the dataflow graph by either splitting a stream into multiple streams or merging streams into a single flow

#### Rolling Aggregations

- A rolling aggregation is an aggregation, such as sum, minimum, and maximum, that is continuously updated for each input event
- Aggregation operations are stateful and combine the current state with the incoming event to produce an updated aggregate value
- The aggregation function must be associative and commutative. Otherwise, the operator would have to store the complete stream history

#### Window Operations

- For some operations which must collect and buffer records to compute their result, eg. join operation, holistic aggregate, median function
- You need to limit the amount of data these operations maintain
- Window operations continuously create finite sets of events called buckets from an unbounded event stream and let us perform computations on these finite sets
- Window policies decide when new buckets are created, which events are assigned to which buckets, and when the contents of a bucket get evaluated
- Common window types:
  - **Tumbling**: windows assign events into nonoverlapping buckets of fixed size. When the window border is passed, all the events are sent to an evaluation function for processing
    - Count-based tumbling windows
    - Time-based tumbling windows
  - **Sliding**: windows assign events into overlapping buckets of fixed size
    - An event might belong to multiple buckets
    - Define sliding windows by providing their length and their slide
  - **Session**: sessions are comprised of a series of events happening in adjacent times followed by a period of inactivity, the length of a session is not defined beforehand but depends on the actual data
    - Session windows group events in sessions based on a session gap value that defines the time of inactivity to consider a session closed
- In practice, you might want to partition a stream into multiple logical streams and define parallel windows
  - In parallel windows, each partition applies the window policies independently of other partitions
- Window operations are closely related to two dominant concepts in stream processing
  - Time semantics
  - State management

## Time Semantics

- Operator semantics should depend on the time when events actually happen and not the time when the application receives the events
- What really defines the amount of events in one minutes is the time of the data itself

### Processing Time

- The time of the local clock on the machine where the operator processing the stream is being executed

### Event Time

- The time when an event in the stream actually happened, based on the timestamp that is attached to the events of the stream (eg. the event creation time)
- Event time correctly places events in a window, reflecting the reality of how things happened
- An event time window computation will yield the same result no matter how fast the stream is processed or when the events arrive at the operator
- When combined with replayable streams, the determinism of timestamps gives you the ability to fast forward the past

### Watermarks

- How do we decide when to trigger an event-time window?
- A watermark is a global progress metric that indicates the point in time when we are confident that no more delayed events will arrive
- When an operator receives a watermark with time `T`, it can assume that no further events with timestamp less than `T` will be received
- **Eager watermarks** -> low latency but provide lower confidence, and we should provide some code to handle late events
- **Relaxed watermarks** -> high confidence but might unnecessarily increase processing latency
- Tracking global progress in a distributed system might be problematic in the presence of straggler tasks, it is crucial that the stream processing system provide some mechanism to deal with events that might arrive after the watermark

