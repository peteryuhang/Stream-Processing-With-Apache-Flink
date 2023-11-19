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

#### Output the Result

- Flink provides a well-maintained collection of stream sinks that can be used to write data to different systems
- It is also possible to implement your own streaming sinks
- There are also applications that do not emit results but keep them internally to serve them via Flink’s queryable state feature

#### Execute

- After completely defined, application can be executed by calling `StreamExecutionEnvironment.execute()`
- Flink programs are executed lazily, only when `execute()` is called does the system trigger the execution of the program
- The constructed plan is translated into a JobGraph and submitted to a JobManager for execution

### Transformations

- Transformations of the DataStream API in 4 categories
  1. Basic transformations are transformations on individual events
  2. KeyedStream transformations are transformations that are applied to events in the context of a key
  3. Multistream transformations merge multiplestreams in to one stream or split one stream into multiple streams
  4. Distribution transformations reorganize stream events

#### Basic Transformations

- Basic transformations process individual events, meaning that each output record was produced from a single input record, eg.
  - Simple value conversions
  - Splitting of records
  - Filtering of records

##### Map

- Specified by calling the `DataStream.map()` method and produces a new `DataStream`
- It passes each incoming event to a user-defined mapper that returns exactly one output event, possibly of a different type

![](./map_transformation.png)

##### Filter

- The filter transformation drops or forwards events of a stream by evaluating a boolean condition on each input event
  - A return value of true preserves the input event and forwards it to the output
  - False results in dropping the event
- Specified by calling the `DataStream.filter()` method and produces a new `DataStream` of the same type as the input DataStream

![](./filter_transformation.png)

##### FlatMap

- Similar to map, but it can produce zero, one, or more output events for each incoming event
- The flatMap transformation is a generalization of filter and map and can be used to implement both those operations

![](./flatmap_transformation.png)


#### KeyedStream Transformations

- All events with the same key access the same state and thus can be processed together

##### KeyBy

- The **keyBy** transformation converts a DataStream into a KeyedStream by specifying a key
  - All events with the same key are processed by the same task of the subsequent operator
- Events with different keys can be processed by the same task, but the keyed state of a task’s function is always accessed in the scope of the current event’s key

![](./keyby_transformation.png)

- The `keyBy()` method receives an argument that specifies the key (or keys) to group by and returns a KeyedStream

##### Rolling Aggregations

- Rolling aggregation transformations are applied on a KeyedStream and produce a DataStream of aggregates, such as sum, minimum, and maximum
- For each incoming event, the operator updates the corresponding aggregate value and emits an event with the updated value
- DataStream API provides the following rolling aggregation methods:
  - sum()
  - min()
  - max()
  - minBy()
  - maxBy()
- It is not possible to combine multiple rolling aggregation methods—only a single rolling aggregate can be computed at a time
- The rolling aggregate operator keeps a state for every key that is processed. Since this state is never cleaned up, you should only apply a rolling aggregations operator on a stream with a bounded key domain

##### Reduce

- The reduce transformation is a generalization of the rolling aggregation
- A reduce transformation does not change the type of the stream
- Similar as rolling aggregations, only use rolling reduce on bounded key domains

#### Multistream Transformations

- Process multiple input streams or emit multiple output streams

##### Union

- `DataStream.union()` method merges two or more DataStreams of the same type and produces a new DataStream of the same type
- The events are merged in a FIFO fashion—the operator does not produce a specific order of events
- The union operator does not perform duplication elimination. Every input event is emitted to the next operator

![](./union_transformation.png)

##### Connect, CoMap, And CoFlatMap

- `DataStream.connect()` method receives a DataStream and returns a **ConnectedStreams** object, which represents the two connected streams
- The ConnectedStreams object provides `map()` and `flatMap()` methods that expect a **CoMapFunction** and **CoFlatMapFunction** as argument respectively
```scala
// IN1: the type of the first input stream
// IN2: the type of the second input stream
// OUT: the type of the output elements
CoMapFunction[IN1, IN2, OUT]
    > map1(IN1): OUT
    > map2(IN2): OUT
// IN1: the type of the first input stream
// IN2: the type of the second input stream
// OUT: the type of the output elements
CoFlatMapFunction[IN1, IN2, OUT]
    > flatMap1(IN1, Collector[OUT]): Unit
    > flatMap2(IN2, Collector[OUT]): Unit
```

- It is not possible to control the order in which the methods of a CoMapFunction or CoFlatMapFunction are called. Instead, a method is called as soon as an event has arrived via the corresponding input
- In order to achieve deterministic transformations, on ConnectedStreams, `connect()` can be combined with `keyBy()` or `broadcast()`

##### Split And Select

- Split is the inverse transformation to the union transformation
- It divides an input stream into two or more output streams of the same type as the input stream
- Split can also be used to filter or replicate events

![](./split_transformation.png)

- `DataStream.split()` method receives an `OutputSelector` that defines how stream elements are assigned to named outputs
- `OutputSelector` defines the `select()` method that is called for each input event and returns a `java.lang.Iterable[String]`
  - The String values that are returned for a record specify the output streams to which the record is routed
- `DataStream.split()` method returns a `SplitStream`, which provides a `select()` method to select one or more streams from the `SplitStream` by specifying the output names
- One restriction of the split transformation is that all outgoing streams are of the same type as the input type


#### Distribution Transformations

- `DataStream` methods that enable users to control partitioning strategies or define their own

- **Random**:
  - Distributes records randomly according to a uniform distribution to the parallel tasks of the following operator
  - Implemented by the `DataStream.shuffle()` method
- **Round-Robin**:
  - Records are evenly distributed to successor tasks in a round-robin fashion
  - Implemented by the `rebalance()` method, and will create communication channels between all sending tasks to all receiving tasks
- **Rescale**:
  - Also distributes events in a round-robin fashion, but only to a subset of successor tasks
  - Implemented by the `rescale()` method, will only create channels from each task to some of the tasks of the downstream operator
  - Offers a way to perform a lightweight load rebalance when the number of sender and receiver tasks is not the same
- **Broadcast**:
  - Implemented by `broadcast()` method, replicaates the input data stream so that all events are sent to all parallel tasks of the downstream operator
- **Global**:
  - Sends all events of the input data stream to the first parallel task of the downstream operator
  - Implemented by `global()` but must be used with care
- **Custom**:
  - You can define your own partitioning strategies by using the `partitionCustom()` method
    - The method receives a `Partitioner` object that implements the partitioning logic
  
### Setting the Parallelism

- The number of parallel tasks of an operator is called the parallelism of the operator
- The parallelism of an operator can be controlled at the level of the execution environment or per individual operator
- By default, the parallelism of all operators of an application is set as the parallelism of the application’s execution environment
  - If the application runs in a local execution environment the parallelism is set to match the number of CPU cores
  - On Flink cluster, the environment parallelism is set to the default parallelism of the cluster unless it is explicitly specified via the submission client
- You can access the default parallelism and also can override the default parallelism

### Types

- Flink uses the concept of type information to represent data types and generate specific serializers, deserializers, and comparators for every data type

#### Supported Data Types

- Primitives
  - All Java and Scala primitive types, such as Integer, String, and Double in Java
- Java and Scala tuples
  - Flink’s Java tuples can have up to 25 fields, with each length is implemented as a separate class—Tuple1, Tuple2, up to Tuple25
  - Tuple fields can be accessed by the name of their public fields —f0, f1, f2, or by position using the `getField(int pos)` method
  - The tuple classes are strongly typed
  - Flink’s Java tuples are mutable, so the values of fields can be reassigned
  ```java
  // DataStream of Tuple2<String, Integer> for Person(name, age)
  DataStream<Tuple2<String, Integer>> persons = env.fromElements(
    Tuple2.of("Adam", 17),
    Tuple2.of("Sarah", 23));
  // filter for persons of age > 18
  persons.filter(p -> p.f1 > 18);

  Tuple2<String, Integer> personTuple = Tuple2.of("Alex", "42");
  Integer age = personTuple.getField(1); // age = 42

  personTuple.f1 = 42; // set the 2nd field to 42
  personTuple.setField(43, 1); // set the 2nd field to 43
  ```
- Scala case classes
- POJOs, including classes generated by Apache Avro
  - Flink accepts a class as a POJO if it satisfies the following conditions:
    - It is a public class
    - It has a public constructor w/o any arguments - a default constructor
    - All fields are public or accessible through getters and setters
      - The getter and setter functions must follow the default naming scheme
        - `Y getX()`
        - `setX(Y x)`
  - Avro-generated classes are automatically identified by Flink and handled as POJOs
- Some special types
  - eg. Java’s ArrayList, HashMap, and Enum types, Hadoop Writable types
- Types that are not specially handled are treated as generic types and serialized using the **Kryo serialization framework**
  - Only use Kryo as a fallback solution since it might not very efficient, and it doesn't provide a good migration path to evolve data types

#### Creating Type Information for Data Types

- The central class in Flink’s type system is **TypeInformation**
- Flink provides two utility classes for Java with static methods to generate a **TypeInformation**
  - The helper class is **org.apache.flink.api.common.typeinfo.Types**:
  ```java
  // TypeInformation for primitive types
  TypeInformation<Integer> intType = Types.INT;
  // TypeInformation for Java Tuples
  TypeInformation<Tuple2<Long, String>> tupleType = Types.TUPLE(Types.LONG, Types.STRING);
  // TypeInformation for POJOs
  TypeInformation<Person> personType = Types.POJO(Person.class);
  ```

#### Explicitly Providing Type Information

- In most cases, Flink can automatically infer types and generate the correct TypeInformation but sometimes the necessary information cannot be extracted, we might need to explicitly provide TypeInformation objects to Flink for some of the data types used in your application
- 2 ways to provide TypeInformation:
  - Extend a function class to explicitly provide the TypeInformation of its return type by implementing the **ResultTypeQueryable** interface
  - In Java API, can use the `returns()` method to explicitly specify
  ```java
  DataStream<Tuple2<String, Integer>> tuples = ...
  DataStream<Person> persons = tuples
    .map(t -> new Person(t.f0, t.f1))
    // provide TypeInformation for the map lambda function's return type
    .returns(Types.POJO(Person.class));
  ```

### Defining Keys and Referencing Fields

- In Flink, keys are defined as functions over the input data, it is not necessary to define data types to hold keys and values, which avoids a lot of boilerplate code

#### Field Positions

- If the data type is a tuple, keys can be defined by simply using the field position of the corresponding tuple element
```scala
val input: DataStream[(Int, String, Long)] = ...
val keyed = input.keyBy(1)
```

- Composite keys consisting of more than one tuple field can also be defined. In this case, the positions are provided as a list, one after the other
```scala
val keyed2 = input.keyBy(1, 2)
```

#### Field Expressions

- Define keys and select fields is by using String- based field expressions

#### Key Selectors

- A KeySelector function extracts a key from an input event
- An advantage of KeySelector functions is that the resulting key is strongly typed due to the generic types of the KeySelector class

### Implementing Functions

#### Function Classes

- A function is implemented by implementing the interface or extending the abstract class
- When a program is submitted for execution, all function objects are serialized using Java serialization and shipped to all parallel tasks of their corresponding operators
- If your function requires a nonserializable object instance, you can either implement it as a rich function and initialize the nonserializable field in the open() method or override the Java serialization and deserialization methods

#### Lambda Functions

- Lambda functions are available for Scala and Java and offer a simple and concise way to implement application logic when no advanced operations such as accessing state and configuration are required

#### Rich Functions

- There is a need to initialize a function before it processes the first record or to retrieve information about the context in which it is executed
- Rich functions can be parameterized just like regular function classes
- The name of a rich function starts with Rich followed by the transformation name — `RichMapFunction`, `RichFlatMapFunction`
- When using a rich function, you can implement two additional methods to the function’s lifecycle:
  - `open()`: Initialization method for the rich function. Called once per task before a transformation method like filter or map is called
    - Typically used for setup work that needs to be done only once
  - `close()`: fFinalization method for the function and it is called once per task after the last call of the transformation method
    - Commonly used for cleanup and releasing resources
- `getRuntimeContext()` provides access to the function's **RuntimeContext**. The **RuntimeContext** can be used to retrieve information such as:
  - Function's parallelism
  - Subtask index
  - Name of the task that executes the function

### Inclding External and Flink Dependencies

