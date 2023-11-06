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