## Setting Up A Development Environment for Apache Flink

### Run Flink Applications in An IDE

- `StreamExecutionEnvironment.execute()`
  - Assembles the dataflow and submits it to a remote JobManager
  - Within the IDE, the whole Flink application is multithreaded and executed within the same JVM process instead of distributed

### Debug Flink Applications in An IDE

- A few things to consider when debugging a Flink application in an IDE:
  - You should be aware that you might debug a multithreaded program
  - The program is executed in a single JVM. Therefore, certain issues, such as classloading issues, cannot be properly debugged
  - Although a program is executed in a single JVM, records are serialized for cross-thread communication and possibly state persistance