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

### Bootstrap a Flink Maven Project

- Run command below to create a Flink Maven Quickstart Scala project:
  ```bash
  mvn archetype:generate                          \
   -DarchetypeGroupId=org.apache.flink            \
   -DarchetypeArtifactId=flink-quickstart-scala   \
   -DarchetypeVersion=1.7.1                       \
   -DgroupId=org.apache.flink.quickstart          \
   -DartifactId=flink-scala-project               \
   -Dversion=0.1                                  \
   -Dpackage=org.apache.flink.quickstart          \
   -DinteractiveMode=false
  ```
- The `src/` folder has the following structure:
  ```
  src/
  └── main
      ├── resources
      │   └── log4j.properties
      └── scala
          └── org
              └── apache
                  └── flink
                      └── quickstart
                          ├── BatchJob.scala
                          └── StreamingJob.scala
  ```