## Time-Based and Window Operators

### Configuring Time Characteristics

- In the DataStream API, you can use the time characteristic to tell Flink how to define time when you are creating windows
- The time characteristic is a property of the `StreamExecutionEnvironment` and it takes the following values:
  - **ProcessingTime**
    - Operators determine the current time of the data stream according to the system clock of the machine where they are being executed
    - In general, using processing time for window operations results in nondeterministic results
    - This setting offers very low latency because processing tasks do not have to wait for watermarks to advance the event time
  - **EventTime**
    - Operators determine the current time by using information from the data itself
    - An event-time window triggers when the watermarks declare that all timestamps for a certain time interval have been received
    - Event-time windows compute deterministic results even when events arrive out of order
  - **IngestionTime**
    - It is a hybrid of EventTime and ProcessingTime
    - The ingestion time of an event is the time it entered the stream processor

### Assigning Timestamps and Generating Watermarks

- Application need to provide 2 important info for Flink operate in event time:
  - Each event must be associated with a timestamp that typically indicates when the event actually happened
  - Event-time stream also need to carry watermarks from which operators infer the current event time
- A watermark tells operators that no more events with a timestamp less than or equal to the watermark are expected
- If a timestamp assigner is used, any existing timestamps and watermarks will be overwritten
- `TimestampAssigner` interface extract timestamps from elements after they have been ingested into the streaming application
- To ensure that event-time operations behave as expected, the assigner should be called before any event-time dependent transformation
- Timestamp assigners are called on a stream of elements and produce a new stream of timestamped elements and watermarks
- Timestamp assigners do not change the data type of a DataStream