## The Architecture of Apache Flink

### System Architecture

- Common challenges that distributed systems need to address:
  - Allocation and management of compute resources in a cluster
  - Coordination, Durable and highly available data storage
  - Failure recovery
- Flink is well integrated with cluster resource managers, such as Apache Mesos, YARN, and Kubernetes, but can also be configured to run as a standalone cluster
- Flink does not provide durable, distributed storage. Instead, it takes advantage of distributed filesystems like HDFS or object stores such as S3
- For leader election in highly available setups, Flink depends on Apache ZooKeeper

#### Components of a Flink Setup

- A Flink setup consists of four different components that work together to execute streaming applications
  - **JobManager**
    - Master process that controls the execution of a single application
    - Works for JobManager
      - Receieves an application for execution (applicaiton consists of JobGraph, JAR file, etc)
      - Converts the JobGraph into a physical dataflow graph (called the ExecutionGraph), which consists of tasks that can be executed in parallel
      - Reuqests the necessary resources (TaskManager slots) from ResourceManager
      - Distributes the tasks of the ExecutionGraph to the TaskManagers that execute them
      - During execution, responsible for all actions that require a central coordination
  - **ResourceManager**
    - Responsible for managing TaskManager slots
    - If the ResourceManager does not have enough slots to fulfill the JobManager’s request, the ResourceManager can talk to a resource provider to provision containers in which TaskManager processes are started
    - Also takes care of terminating idle TaskManagers to free compute resources
  - **TaskManager**
    - Worker processes of Flink
    - Each TaskManager provides a certain number of slots. The number of slots limits the number of tasks a TaskManager can execute
    - After it has been started, a TaskManager registers its slots to the ResourceManager
    - When instructed by the ResourceManager, the TaskManager offers one or more of its slots to a JobManager
    - The JobManager can then assign tasks to the slots to execute them
    - During execution, a TaskManager exchanges data with other TaskManagers that run tasks of the same application
  - **Dispatcher**
    - Serve as an HTTP entry point to clusters that are behind a firewall
    - Runs across job executions and provides a REST interface to submit applications for execution
    - Once an application is submitted for execution, it starts a JobManager and hands the application over
    - Also runs a web dashboard to provide information about job executions
    - Dispatcher might not be required in some scenarios

![](./component_interactions.png)

#### Application Deployment

- **Framwork style**
  - Flink application are packaged into a JAR file and submitted by a client to a running service
  - If application was submitted to JobManager, it immediately starts to execute the application
  - If application was submitted to Dispatcher or YARN ResourceManager, it will spin up a JobManager and hand over the application
  - Traditional approach of submitting an application (or query) via a client to a running service
- **Libaray style**
  - Flink application is bundled in an application specific container image
  - When a container is started from the image, it automatically launches the ResourceManager and JobManager and submits the bundled job for execution
  - Common for microservices architectures

#### Task Execution

- A TaskManager can execute several tasks at the same time
- These tasks can be subtasks of the same operator (data parallelism), a different operator (task parallelism), or even from a different application (job parallelism)
- A processing slot can execute one slice of an application—one parallel task of each operator of the application

![](./operator_tasks_and_slots.png)

- One the Figure above:
  - Left-hand side is the JobGraph (nonparallel representation of an application)
- TaskManager can efficiently exchange data within the the same process and without accessing the network
- A TaskManager executes its tasks multithreaded in the same JVM process
  - Threads are more lightweight than separate processes and have lower communication costs but do not strictly isolate tasks from each other
  - A single misbehaving task can kill a whole TaskManager process and all tasks that run on it
  - By configuring only a single slot per TaskManager, you can isolate applications across TaskManagers

#### Highly Available Setup

##### TaskManager Failures

- If one of the TaskManagers fails, the JobManager will ask the ResourceManager to provide more processing slots
- The application’s restart strategy determines how often the JobManager restarts the application and how long it waits between restart attempts

##### JobManager Failures

- Flink supports a high-availability mode that migrates the responsibility and metadata for a job to another JobManager in case the original JobManager disappears
- Flink’s high-availability mode is based on Apache ZooKeeper
- When operating in high-availability mode, the JobManager writes the JobGraph and all required metadata, such as the application’s JAR file, into a remote persistent storage system
- In addition, the JobManager writes a pointer to the storage location into ZooKeeper’s datastore

![](./highly_available_flink_setup.png)

- When a JobManager fails, all tasks that belong to its application are automatically cancelled. A new JobManager that taks over the work and perform steps:
  1. It requests the storage locations from ZooKeeper to fetch the JobGraph, the JAR file, and the state handles of the last checkpoint of the application from the remote storage
  2. It requests processing slots from the ResourceManager to continue executing the application
  3. It restarts the application and resets the state of all its tasks to the last completed checkpoint
- Flink does not provide tooling to restart failed processes when running in a standalone cluster
  - It can be useful to run standby JobManagers and TaskManagers that can take over the work of failed processes

### Data Transfer in Flink

- The network component of a TaskManager collects records in buffers before they are shipped, i.e., records are not shipped one by one but batched into buffers
- Shipping records in buffers does imply that Flink’s processing model is based on microbatches
- Each TaskManager has a pool of network buffers (by default 32 KB in size) to send and receive datas
- If the sender and receiver tasks run in separate TaskManager processes, they communicate via the network stack of the operating system

![](./data_transfer_between_taskmanager.png)

- Each of the 4 sender tasks needs at least 4 network buffers to send data to each of the receiver tasks and each receiver task requires at least 4 buffers to receive data
- Buffers that need to be sent to the other TaskManager are multiplexed over the same network connection
- With a shuffle or broadcast connection, each sending task needs a buffer for each receiving task; the number of required buffers is quadratic to the number of tasks of the involved operators

#### Credit-Based Flow Control

- A receiving task grants some credit to a sending task, the number of network buffers that are reserved to receive its data
- Once a sender receives a credit notification, it ships as many buffers as it was granted and the size of its backlog
- The receiver processes the shipped data with the reserved buffers and uses the sender’s backlog size to prioritize the next credit grants for all its connected senders
- Credit-based flow control reduces latency because senders can ship data as soon as the receiver has enough resources to accept it
- it is an effective mechanism to distribute network resources in the case of skewed data distributions because credit is granted based on the size of the senders’ backlog

#### Task Chaining

- In order to satisfy the requirements for task chaining, two or more operators must be configured with the same parallelism and connected by local forward channels

![](./task_chaining1.png)

- How the pipeline is executed with task chaining:
  - The functions of the operators are fused into a single task that is executed by a single thread
  - Records that are produced by a function are separately handed over to the next function with a simple method call
  - No serialization and communication costs for passing records

- Execute a pipeline without chaining:

![](./task_chaining2.png)

- Task chaining is enabled by default in Flink

### Event-Time Processing

#### Timestamps

- All records that are processed by a Flink event-time streaming application must be accompanied by a timestamp
- When Flink processes a data stream in event-time mode, it evaluates time-based operators based on the timestamps of records
- Flink encodes timestamps as 16-byte Long values and attaches them as metadata to records

#### Watermarks

- In addition to record timestamps, a Flink event-time application must also provide watermarks
- Time-based operators use this time to trigger computations and make progress
- Watermarks flow in a stream of regular records with annotated timestamps

![](./timestamp_with_watermarks.png)

- 2 basic properties of watermarks:
  1. They must be monotonically increasing to ensure the event-time clocks of tasks are progressing and not going backward
  2. They are related to record timestamps. A watermark with a timestamp `T` indicates that all subsequent records should have timestamps > `T`

- **Late records**: a record that violates the watermark property and has smaller timestamps than a previously received watermark
- Watermarks allow an application to control result completeness and latency, it comes with trade-off
  - Very tight watermark result in low processing latency but poor result completeness
  - Very conservative watermarks increase processing latency but improve result completeness

#### Watermark Propagation and Event Time

- Flink implements watermarks as special records that are received and emitted by operator tasks
- When a task receives a watermark, the following actions take place
  1. The task updates its internal event-time clock based on the watermark’s timestamp
  2. The task’s time service identifies all timers with a time smaller than the updated event time. For each expired timer, the task invokes a callback function that can perform a computation and emit records
  3. The task emits a watermark with the updated event time
- When tasks receives a watermark from a input partition:
  1. It updates the respective partition watermark to be the maximum of the received value and the current value
  2. Updates its event-time clock to be the minimum of all partition watermarks
  3. If the event-time clock advances, the task processes all triggered timers and finally broadcasts its new event time to all downstream tasks by emitting a corresponding watermark to all connected output partitions
- This algorithm is relies on the fact that all partitions continuously provide increasing watermarks
- If any partition have problem, the event-time clock of a task will not advance and the timers of the task will not trigger
  - A similar effect appears for operators with two input streams whose watermarks significantly diverge, the faster stream are buffered in state until the event-time clock allows processing them

#### Timestamp Assignment and Watermark Generation

- A Flink DataStream application can assign timestamps and generate watermarks to a stream in three ways
  1. **At the source - sourceFunction**: 
    - Generated by sourceFunction when a stream is ingested into an application
    - If a source function (temporarily) does not emit anymore watermarks, it can declare itself idle
    - Flink will exclude stream partitions produced by idle source functions from the watermark computation of subsequent operators
  2. **Periodic assigner - AssignerWithPeriodicWatermarks**:
    - Extracts a timestamp from each record and is periodically queried for the current watermark
  3. **Punctuated assigner - AssignerWithPunctuatedWatermarks**:
    - Extracts a timestamp from each record
    - It can be used to generate watermarks that are encoded in special input records
    - This function can extract a watermark from each record
- User-defined timestamp assignment functions are usually applied as close to a source operator as possible because it can be very difficult to reason about the order of records and their timestamps after they have been processed by an operator

### State Management

- Flink treats all states—regardless of built-in or user-defined operators—the same
- All data maintained by a task and used to compute the results of a function belong to the state of the task

![](./stateful_stream_processing_task.png)

- Some challengings (all these are taken care of by Flink):
  - Handle of very large states, possibly exceeding memory
  - Ensure that no state is lost in case of failures
- In Flink, state is always associated with a specific operator
- In order to make Flink’s runtime aware of the state of an operator, the operator needs to register its state

#### Operator State

- Scoped to an operator task
- All records processed by the same parallel task have access to the same state
- Flink offers three primitives for operator state:
  1. **List state**: Represents state as a list of entries
  2. **Union list state**: Similar as list state, but will be different during restored time
  3. **Broadcast state**: Designed for the special case where the state of each task of an operator is identical

![](./operator_state.png)

#### Keyed State

- Keyed state is maintained and accessed with respect to a key defined in the records of an operator’s input stream
- Flink maintains one state instance per key value and partitions all records with the same key to the operator task that maintains the state for this key
- In the end, all records with the same key access the same state
- Flink provides different primitives for keyed state that determine the type of the value stored for each key in this distributed key-value map:
  1. **Value state**: Stores a single value of arbitrary type per key. Complex data structures can also be stored as value state
  2. **List state**: Stores a list of values per key. The list entries can be of arbitrary type
  3. **Map state**: Stores a key-value map per key. The key and value of the map can be of arbitrary type

![](./keyed_state.png)

#### State Backends

- How exactly the state is stored, accessed, and maintained is determined by a pluggable component that is called a **state backend**
- A state backend is responsible for two things
  - **local state management**:
    - State backend stores all keyed states and ensures that all accesses are correctly scoped to the current key
    - Manage keyed state as objects stored in in-memory data structures on the JVM heap (fast, but limited by the size of the memory)
    - Another state backend serializes state objects and puts them into RocksDB (slow, but might grow very large)
  - **checkpointing state to a remote location**:
    - State backend takes care of checkpointing the state of a task to a remote and persistent storage
    - State backends differ in how state is checkpointed

#### Scaling Stateful Operators

- Operators with keyed state are scaled by repartitioning keys to fewer or more tasks
- Flink does not redistribute individual keys, instead, Flink organizes keys in so-called key groups

![](./scaling_an_operator_with_keyed_state.png)

- Operators with operator list state are scaled by redistributing the list entries
- The list entries of all parallel operator tasks are collected and evenly redistributed to a smaller or larger number of tasks

![](./scaling_an_operator_with_list_state.png)

- Operators with operator union list state are scaled by broadcasting the full list of state entries to each task
- The task can then choose which entries to use and which to discard

![](./scaling_an_operator_with_union_list_state.png)

- Operators with operator broadcast state are scaled up by copying the state to new tasks
- This works because broadcasting state ensures that all tasks have the same state

![](./scaling_an_operator_with_broadcast_state.png)

### Checkpoints, Savepoints, and State Recovery

#### Consistent Checkpoints

- A consistent checkpoint of a stateful streaming application is a copy of the state of each of its tasks at a point when all tasks have processed exactly the same input
- Steps of naive algorithm (Flink does not implement this mechanism):
  1. Pause the ingestion of all input streams
  2. Wait for all in-flight data to be completely processed, meaning all tasks have processed all their input data
  3. Take a checkpoint by copying the state of each task to a remote, persistent storage. The checkpoint is complete when all tasks have finished their copies
  4. Resume the ingestion of all streams

#### Recovery from a Consistent Checkpoint

- An application is recovered in 3 steps:
  1. Restart the whole application
  2. Reset the states of all stateful tasks to the latest checkpoint
  3. Resume the processing of all task
- An application can only be operated under exactly-once state consistency if all input streams are consumed by resettable data sources (eg. Kafka)
- Flink’s checkpointing and recovery mechanism only resets the internal state of a streaming application, some result records might be emitted multiple times to downstream systems
  - For some storage systems, Flink provides sink functions that feature exactly-once output, eg. by committing emitted records on checkpoint completion

#### Flink's Checkpointing Algorithm

- Flink implements checkpointing based on the Chandy–Lamport algorithm for distributed snapshots
- This algorithm use a special type of record called a **checkpoint barrier**
  - All state modifications due to records that precede a barrier are included in the barrier’s checkpoint
  - All modifications due to records that follow the barrier are included in a later checkpoint
- Example for showing the algorithm
  ![](./checkpoint_algorithm_1.png)
- A checkpoint is initiated by the JobManager by sending a message with a new checkpoint ID to each data source task
  ![](./checkpoint_algorithm_2.png)
- The source task's state backend send the notification to following task once its state checkpoint is complete and acknowledges the checkpoint at the JobManager
  ![](./checkpoint_algorithm_3.png)
- When a task receives a barrier for a new checkpoint, it waits for the arrival of barriers from all its input partitions for the checkpoint (**barrier alignment**)
- While it is waiting, it continues processing records from stream partitions that did not provide a barrier yet
- Records that arrive on partitions that forwarded a barrier already cannot be processed and are buffered
  ![](./checkpoint_algorithm_4.png)
- As soon as a task has received barriers from all its input partitions, it initiates a checkpoint at the state backend and broadcasts the checkpoint barrier to all of its downstream connected tasks
  ![](./checkpoint_algorithm_5.png)
- Once all checkpoint barriers have been emitted, the task starts to process the buffered records
- After all buffered records have been emitted, the task continues processing its input streams
  ![](./checkpoint_algorithm_6.png)
- Eventually, the checkpoint barriers arrive at a sink task
- When a sink task receives a barrier, it performs a barrier alignment, checkpoints its own state, and acknowledges the reception of the barrier to the JobManager
- The JobManager records the checkpoint of an application as completed once it has received a checkpoint acknowledgement from all tasks of the application
  ![](./checkpoint_algorithm_7.png)

#### Performace Implications of Checkpointing

- Checkpoints can increase the processing latency of an application, Flink implements some tweaks (implemented on the state backend):
  - **asynchronous checkpoints**: create the local copy, then continues its regular processing, background thread asynchronously copies the local snapshot to the remote storage
  - **incremental checkpointing** in RocksDB to reduces the amount of data to transfer
  - Tweak the barrier alignment step:
    - Flink can be configured to process all arriving records during buffer alignment instead of buffering those for which the barrier has already arrived
    - Once all barriers for a checkpoint have arrived, the operator checkpoints the state, which might now also include modifications caused by records that would usually belong to the next checkpoint
    - In case of a failure, these records will be processed again, which means the checkpoint provides at-least-once instead of exactly-once consistency guarantees

#### Savepoints

- Checkpoints are periodically taken and automatically discarded according to a configurable policy
- In principle, savepoints are created using the same algorithm as checkpoints and hence are basically checkpoints with some additional metadata
- Flink does not automatically take a savepoint, so a user (or external scheduler) has to explicitly trigger its creation
- Flink also does not automatically clean up savepoints

#### Using Savepoints

- Given an application and a compatible savepoint, you can start the application from the savepoint
- Usage examples:
  - Start a different but compatible application from a savepoint
  - Start the same application with a different parallelism and scale the application out or in
  - Start the same application on a different cluster
  - Use a savepoint to pause an application and resume it later
  - You can also just take a savepoint to version and archive the state of an application

#### Starting An Application From A Savepoint

- A typical application consists of multiple states that are distributed across multiple operator tasks that can run on different TaskManager processes
- When a savepoint is taken, the states of all tasks are copied to a persistent storage location
- The state copies in the savepoint are organized by an operator identifier and a state name
![](./savepoint.png)
- A state in the savepoint can only be mapped to the application if it contains an operator with a corresponding identifier and state name