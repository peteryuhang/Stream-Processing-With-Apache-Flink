## Traditional Data Infrastructures

### Transactional Processing

- Cause problems when applications need to evolve or scale
  - Multiple application might work on the same data representation or share the same infrastructure, change the schema of a table or scaling a database system requires careful planning and lots of effort
  - Microservice can help resolve and make this be better

### Analytical Processing

![](./analytics_processing.png){: width="372" height="89" }

- ETL: Extract Transform Load
  - Process used to copy data from transactional databases to data warehouse

## Stateful Stream Processing

- Apache Flink stores the application state locally in memory or in an embedded database
- Since Flink is a distributed system, the local state needs to be protected against failures to avoid data loss in case of application or machine failure. Flink guarantees this by periodically writing a consistent checkpoint of the application state to a remote and durable storage

![](./flink_stage.png){: width="372" height="89" }