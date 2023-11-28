## Reading from and Writing to External Systems

### Application Consistency Guaratees

- The source and sink connectors of an application need to be integrated with Flink's checkpoints and recovery mechanism and provide certain properties to be able to give meaningful guarantees
- Source connector of the application needs to be able to set its read positions to a previously checkpointed position
- Resettable source connectors guarantees that an application will not lose any data, but are not sufficient to provide end-to-end exactly-once guaranteess
- An application that aims to provide end-to-end exactly-once guarantees requires special sink connectors

