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

