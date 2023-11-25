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