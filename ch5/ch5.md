## The DataStream API (v1.7)

### Hello, Flink!

- To structure a typical Flink streaming application:
  1. Set up the execution environment
  2. Read one or more streams from data sources
  3. Apply streaming transformations to implement the application logic
  4. Optionally output the result to one or more data sinks
  5. Execute the program

#### Set Up the Execution Environment

- The execution environment determines whether the program is running on a local machine or on a cluster
- In the DataStream API, the execution environment of an application is represented by the `StreamExecutionEnvironment`
  - `StreamExecutionEnvironment.getExecutionEnvironment()` returns a local or remote environment
  - It is also possible to explicitly create local or remote execution environments:
    - `StreamExecutionEnvironment.createLocalEnvironment()` for local
    - `StreamExecutionEnvironment.createRemoteEnvironment()` for remote
  - The execution environment offers more configuration options, such as setting the program parallelism and enabling fault tolerance
- `env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)` to instruct our program to interpret time semantics using event time

#### Read an Input Stream

- Data streams can be ingested from sources such as message queues or files, or also be generated on the fly
- We create an initial DataStream of type SensorReading:
  ```java
  // ingest sensor stream
  // SensorReading contains the sensor ID, a timestamp denoting when the measurement was taken, and the measured temperature
  DataStream<SensorReading> sensorData = env
      // SensorSource generates random temperature readings
      .addSource(new SensorSource())
      // assign timestamps and watermarks which are required for event time
      .assignTimestampsAndWatermarks(new SensorTimeAssigner());
  ```

#### Apply Transformations

- The logic of an application is defined by chaining transformations

```java
DataStream<SensorReading> avgTemp = sensorData
    // convert Fahrenheit to Celsius using and inlined map function
    .map(r -> new SensorReading(r.id, r.timestamp, (r.temperature - 32) * (5.0 / 9.0)))
    // organize stream by sensor
    .keyBy(r -> r.id)
    // group readings in 1 second windows
    .timeWindow(Time.seconds(1))
    // compute average temperature using a user-defined function
    .apply(new TemperatureAverager());
```
