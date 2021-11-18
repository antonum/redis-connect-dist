# redis-connect-oracle

redis-connect-oracle is a Redis Connect connector for capturing changes (INSERT, UPDATE and DELETE) from Oracle Database (source) and writing them to a Redis Enterprise database (Target). The connector uses [Oracle LogMiner](https://docs.oracle.com/cd/B19306_01/server.102/b14215/logminer.htm#i1010243) to read the database redo log.

<p>
The first time redis-connect-oracle connects to a Oracle database, it reads a consistent snapshot of all of the schemas.
When that snapshot is complete, the connector continuously streams the changes that were committed to Oracle and generates a corresponding insert, update or delete event.
All of the events for each tables are recorded in a separate Redis data structure or module of your choice, where they can be easily consumed by applications and services.

| ℹ️ |
|:---------------------------|
| Quick Start: Follow the [demo](demo)|
| K8s Setup: Follow the [k8s-docs](k8s-docs)|