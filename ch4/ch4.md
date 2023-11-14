## Setting Up A Development Environment for Apache Flink

### Run and Debug Flink Application in An IDE

- `StreamExecutionEnvironment.execute()`
  - Assembles the dataflow and submits it to a remote JobManager
  - Within the IDE, the whole Flink application is multithreaded and executed within the same JVM process instead of distributed