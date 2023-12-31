## Stateful Operators and Applications

- This chapter focuses on the implementation of stateful user-defined functions and discusses the performance and robustness of stateful applications

### Implementing Stateful Functions

#### Declaring Keyed State at RuntimeContext

- Keyed state is very similar to a distributed key-value map
  - Each parallel instance of the function is responsible for a subrange of the key domain and maintains the corresponding state instances
- Keyed state can only be used by functions that are applied on a KeyedStream
  - A KeyedStream is constructed by calling the `DataStream.keyBy()`
- A **state primitive** defines the structure of the state for an individual key. The following state primitives are supported by Flink
  - `ValueState[T]` holds a single value of type T
  - `ListState[T]` holds a list of elements of type T
    - It is not possible to remove individual elements from ListState, but it can be updated
  - `MapState[K, V]` holds a map of keys and values
  - `ReducingState[T]` offers the same methods as `ListState[T]` (except for `addAll()` and `update()`)
    - Instead of appending values to a list, `ReducingState.add()` immediately aggregates value using a ReduceFunction
  - `AggregatingState[I, O]` behaves similar to `ReducingState`, but it use more general `AggregateFunction` to aggregate values
  - All state primitives can be cleared by calling `State.clear()`

#### Implementing Operator List State with the ListCheckpointed Interface

- A function can work with operator list state by implementing the **ListCheckpointed** interface
- The interface provide 2 methods:
  ```scala
  // returns a snapshot the state of the function as a list
  snapshotState(checkpointId: Long, timestamp: Long): java.util.List[T]

  // restores the state of the function from the provided list
  restoreState(java.util.List[T] state): Unit
  ```

#### Using Connected Broadcast State

- A common requirement in streaming applications is to distribute the same information to all parallel instances of a function and maintain it as recoverable state
- Broadcast state can be combined with a regular **DataStream** or **KeyedStream**
- Broadcasted events might not arrive in deterministic order

#### Using the CheckpointedFunction Interface

- `CheckpointedFunction` provides hooks to register and maintain keyed state and operator state and is the only interface that gives access to operator list union state

#### Receiving Notifications About Completed Checkpoints

- As discussed before, a checkpoint is only successful if all operator tasks successfully checkpointed their states to the checkpoint storage
  - Only JobManager can determine whether a checkpoint is successful or not
- Operators that need to be notified about completed checkpoints can implement the **CheckpointListener** interface

### Enabling Failure Recovery for Stateful Application

- Applications need to explicitly enable the periodic checkpointing mechanism via the StreamExecutionEnvironment

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
// set checkpointing interval to 10 seconds (10000 milliseconds)
env.enableCheckpointing(10000L)
```

- A shorter checkpointing interval causes higher overhead during regular processing but can enable faster recovery because less data needs to be reprocessed

### Ensuring the Maintainability of Stateful Applications

- It is important that application state can be migrated to a new version of the application or be redistributed to more or fewer operator tasks
- Flink features savepoints to maintain applications and their states
- It requires that all stateful operators of the initial version of an application specify two parameters to ensure the application can be properly maintained in the future

#### Specifying Unique Operator Identifiers

- Savepoint state can only be restored to an operator of a started application if their identifiers are identical

#### Defining the Maximum Parallelism of Keyed State Operators

- The maximum parallelism parameter of an operator defines the number of key groups into which the keyed state of the operator is split
- The maximum parallelism can be set for all operators of an application via the **StreamExecutionEnvironment** or per operator using the `setMaxParallelism()` method


### Performance and Robustness of Stateful Applications

#### Choosing a State Backend

- The state backend is responsible for storing the local state of each task instance and persisting it to remote storage when a checkpoint is taken
- Flink offers 3 state backends:
  - **MemoryStateBackend**
    - Stores state as regular objects on the heap of the TaskManager JVM process
    - MemoryStateBackend is only recommended for development and debugging purposes
  - **FsStateBackend**
    - Stores the local state on the TaskManager’s JVM heap
    - The difference is it writes the state to a remote and persistent file system instead of JobManager's volatile memory
  - **RocksDBStateBackend**
    - Stores all state into local RocksDB instances
    - RocksDB is an embedded key-value store that persists data to the local disk
    - RocksDBStateBackend is a good choice for applications with very large state
    - Reading and writing data to disk and the overhead of de/serializing objects result in lower read and write performance compared to maintaining state on the heap

#### Choosing a State Primitive

- For state backends that de/serialize state objects when reading or writing, such as **RocksDBStateBackend**, the choice of the state primitive (ValueState, ListState, or MapState) can have a major impact on the performance of an application


#### Preventing Leaking State

- A common reason for growing state is keyed state on an evolving key domain
- A solution for this problem is to remove the state of expired keys

### Evolving Stateful Applications

- A running application needs to be replaced by an updated version usually without losing the state of the application

#### Updating an Application without Modifying Existing State

- If an application is updated without removing or changing existing state, it is always savepoint compatible and can be started from a savepoint of an earlier version
- If you add a new stateful operator to the application or a new state to an existing operator, the state will be initialized as empty when the application is started from a savepoint

#### Removing State from an Application

- When the new version of the application is started from a savepoint of the previous version, the savepoint contains state that cannot be mapped to the restarted application
- By default, Flink will not start applications that do not restore all states that are contained in a savepoint to avoid losing the state in the savepoint
  - It is possible to disable this safety check

#### Modifying the State of an Operator

- There are 2 ways state can be modified:
  - By changing the data type of a state (Support)
  - By changing the type of a state primitive (Not support)

### Queryable State

- Flink features **queryable state** to address use cases that usually would require an external datastore to share data
- In Flink, any keyed state can be exposed to external applications as queryable state and act as a read-only key-value store
- Only key point queries are supported. It is not possible to request key ranges or even run more complex queries

#### Architecture and Enabling Queryable State

- Flink's queryable state service consists of 3 processes:
  - `QueryableStateClient` used by an external application to submit queries and retrieve results
  - `QueryableStateClientProxy` accepts and serves client requests
    - Each TaskManager runs a client proxy
  - `QueryableStateServer` serves the requests of a client proxy
    - Each TaskManager runs a state server that fetches the state of a queried key from the local state backend and returns it to the requesting client proxy

![](./arch_of_queryable_state_service.png)

- In order to enable the queryable state service in a Flink setup, you need to add the **flink-queryable-state-runtime** JAR file to the classpath of the TaskManager process

#### Exposing Queryable State

- Define a function with keyed state and make the state queryable by calling the `setQueryable(String)` method on the StateDescriptor before obtaining the state handle
- An application with a function that has a queryable state is executed just like any other application

#### Querying State from External Applications

- Any JVM-based application can query the queryable state of a running Flink application by using **QueryableStateClient**
- By default, the client proxy listens on port 9067, but the port can be configured in the `./conf/flink-conf.yaml` file
- Once you obtain a state client, you can query the state of an application by calling the `getKvState()` method