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

#### Assigning Timestamps and Generating Watermarks

- Application need to provide 2 important info for Flink operate in event time:
  - Each event must be associated with a timestamp that typically indicates when the event actually happened
  - Event-time stream also need to carry watermarks from which operators infer the current event time
- A watermark tells operators that no more events with a timestamp less than or equal to the watermark are expected
- If a timestamp assigner is used, any existing timestamps and watermarks will be overwritten
- `TimestampAssigner` interface extract timestamps from elements after they have been ingested into the streaming application
- To ensure that event-time operations behave as expected, the assigner should be called before any event-time dependent transformation
- Timestamp assigners are called on a stream of elements and produce a new stream of timestamped elements and watermarks
- Timestamp assigners do not change the data type of a DataStream

##### Assigner With Periodic Watermarks

- Instruct the system to emit watermarks and advance the event time in fixed intervals of machine time
- Default interval is set to two hundred milliseconds, but it can be configured using the `ExecutionConfig.setAutoWatermarkInterval()`
- If your input elements have timestamps that are monotonically increasing, you can use the shortcut method `assignAscendingTimeStamps`
- When you know the maximum lateness that you will encounter in the input stream, can use `BoundedOutOfOrdernessTimeStampExtractor`

##### Assigner With Punctuated Watermarks

- Flink provides the `AssignerWithPunctuatedWatermarks` interface for watermarks can be defined based on some other property of the input elements
  - It defines the `checkAndGetNextWatermark()` method, which is called for each event right after `extractTimestamp()`
  - `checkAndGetNextWatermark()` can decide to generate a new watermark or not
  - A new watermark is emitted if the method returns a nonnull watermark that is larger than the latest emitted watermark

#### Watermarks, Latency, and Completeness

- Watermarks are used to balance latency and result completeness
- Watermarks control how long to wait for data to arrive before performing a computation
- The reality is that we can never have perfect watermarks because that would mean we are always certain there are no delayed records
- The latency/completeness tradeoff is a fundamental characteristic of stream processing applications

### Process Functions

- Process functions can access record timestamps and watermarks and register timers that trigger at a specific time in the future
- Process functions are commonly used to build event-driven applications and to implement custom logic for which predefined windows and transformations might not be suitable
- Flink provides eight different process functions:
  - **ProcessFunction**
  - **KeyedProcessFunction**
  - **CoProcessFunction**
  - **ProcessJoinFunction**
  - **BroadcastProcessFunction**
  - **KeyedBroadcastProcessFunction**
  - **ProcessWindowFunction**
  - **ProcessAllWindowFunction**
- All process functions implement the RichFunction interface and hence offer `open()`, `close()`, and `getRuntimeContext()` methods
- **KeyedProcessFunction** provides the following 2 methods:
  - `processElement()`
    - Called for each record of the stream
  - `onTimer()`
    - Callback function that is invoked when a previously registered timer triggers

#### TimerService and Timers

- The TimerService of the Context and OnTimerContext objects offers the following methods
  - `currentProcessingTime()`: returns the current processing time
  - `currentWatermark()`: returns the timestamp of the current watermark
  - `registerProcessingTimeTimer()`: registers a processing time timer for the current key
  - `registerEventTimeTimer()`: registers an event-time timer for the current key. The timer will fire when the watermark is updated to a timestamp that is equal to or larger than the timer’s timestamp
  - `deleteProcessingTimeTimer()`: deletes a processing-time timer that was previously registered for the current key
  - `deleteEventTimeTimer()`: deletes an event-time timer that was previously registered for the current key
- When a timer fires, the onTimer() callback function is called
- Timers can only be registered on keyed streams

#### Emitting to Side Outputs

- Side outputs are a feature of process functions to emit multiple streams from a function with possibly different types
- Process functions can emit a record to one or more side outputs via the Context object

#### CoProcessFunction

- A CoProcessFunction offers a transformation method for each input

### Window Operators

- Windows enable transformations such as aggregations on bounded intervals of an unbounded stream

#### Defining Window Operators

- Window operators on keyed windows are evaluated in parallel, and nonkeyed windows are processed in a single thread
- To create a window operator, you need to specify two window components:
  - A **window assigner** that determines how the elements of the input stream are grouped into windows. It produces a WindowedStream (or AllWindowedStream if applied on a nonkeyed DataStream)
  - A **window function** that is applied on a WindowedStream (or AllWindowedStream) and processes the elements that are assigned to a window


#### Built-in Window Assigners

- Time-based window assigners assign an element based on its event-time timestamp or the current processing time to windows
- Time windows have a start and an end timestamp
- All built-in window assigners provide a default trigger that triggers the evaluation of a window once the (processing or event) time passes the end of the window
- It is important to note that a window is created when the first element is assigned to it. Flink will never evaluate empty windows
- Flink’s built-in window assigners create windows of type **TimeWindow**
  - This window type essentially represents a time interval between the two timestamps, where start is inclusive and end is exclusive
  - It exposes methods to retrieve the window boundaries, to check whether windows intersect, and to merge overlapping windows

##### Tumbling Windows

- A tumbling window assigner places elements into nonoverlapping, fixed-size windows

![](./tumbling_window.png)

- The DataStream API provide 2 tumbling window assigner
  - `TumblingEventTimeWindows`
  - `TumblingProcessingTimeWindows`

- A tumbling window assigner receives one parameter, the window size in time units; this can bespecified using the `of(Time size)` method of the assigner

##### Sliding Windows

- The sliding window assigner assigns elements to fixed-sized windows that are shifted by a specified slide interval

![](./sliding_window.png)

- For a sliding window, you have to specify a window size and a slide interval that defines how frequently a new window is started
  - When the slide interval is smaller than the window size, the windows overlap and elements can be assigned to more than one window
  - If the slide is larger than the window size, some elements might not be assigned to any window and hence may be dropped

##### Session Windows

- A session window assigner places elements into nonoverlapping windows of activity of varying size
- The boundaries of session windows are defined by gaps of inactivity, time intervals in which no record is received

![](./session_window.png)

#### Applying Functions on Windows

- Window functions define the computation that is performed on the element of a window
- There are 2 types of functions that can be applied on a window:
  - **Incremental aggregation functions**:
    - Hold and update a single value as window state, and eventually emit the aggregated value as a result
    - eg. `ReduceFunction`, `AggregateFunction`
  - **Full window functions**:
    - Collect all elements of a window and iterate over the list of all collected elements when they are evaluated
    - `ProcessWindowFunction`

##### Reduce Function

- When a new element is received, the `ReduceFunction` is called with the new element and the current value that is read from the window’s state
- The window’s state is replaced by the ReduceFunction’s result
- The input and output type must be the same

##### Aggregate Function

- Similar to `ReduceFunction`, but `AggregateFunction` is much more flexible
- Interface of the `AggregateFunction`:
  ```scala
  public interface AggregateFunction<IN, ACC, OUT> extends Function, Serializable {
    // create a new accumulator to start a new aggregate.
    ACC createAccumulator();

    // add an input element to the accumulator and return the accumulator.
    ACC add(IN value, ACC accumulator);

    // compute the result from the accumulator and return it.
    OUT getResult(ACC accumulator);

    // merge two accumulators and return the result.
    ACC merge(ACC a, ACC b);
  }
  ```
- In contrast to the `ReduceFunction`, the intermediate data type and the output type do not depend on the input type

##### Process Window Function

- Perform arbitrary computations on the contents of a window
- Interface of the `ProcessWindowFunction`:
  ```scala
  public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> extends AbstractRichFunction {
    // Evaluates the window
    void process(KEY key, Context ctx, Iterable<IN> vals, Collector<OUT> out) throws Exception;
    // Deletes any custom per-window state when the window is purged
    public void clear(Context ctx) throws Exception {}
    // The context holding window metadata
    public abstract class Context implements Serializable {
      // Returns the metadata of the window
      public abstract W window();
      // Returns the current processing time
      public abstract long currentProcessingTime();
      // Returns the current event-time watermark
      public abstract long currentWatermark();
      // State accessor for per-window state
      public abstract KeyedStateStore windowState();
      // State accessor for per-key global state
      public abstract KeyedStateStore globalState();
      // Emits a record to the side output identified by the OutputTag.
      public abstract <X> void output(OutputTag<X> outputTag, X value);
    }
  }
  ```

##### Incremental Aggregation And Process Window Function

- If you have incremental aggregation logic but also need access to window metadata, you can combine a ReduceFunction or AggregateFunction, which perform incremental aggregation, with a ProcessWindowFunction, which provides access to more functionality
- eg.
  ```scala
  input
    .keyBy(...)
    .timeWindow(...)
    .reduce(
      incrAggregator: ReduceFunction[IN],
      function: ProcessWindowFunction[IN, OUT, K, W])

  input
    .keyBy(...)
    .timeWindow(...)
    .aggregate(
      incrAggregator: AggregateFunction[IN, ACC, V],
      windowFunction: ProcessWindowFunction[V, OUT, K, W])
  ```

