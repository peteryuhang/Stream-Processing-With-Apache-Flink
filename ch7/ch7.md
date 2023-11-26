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