<p align="left"><img src="/docs/images/RedisConnect.svg" alt="Redis Connect" height="100px"></p>

# Redis Connect (Continuous Replication and Initial Load to Redis Enterprise)

- [Redis Connect Overview](#technical-overview)
  * [Using redis-connect-cassandra](connectors/cassandra#redis-connect-cassandra)
  * [Using redis-connect-db2](connectors/db2#redis-connect-db2)
  * [Using redis-connect-gemfire](connectors/gemfire#redis-connect-gemfire)
  * [Using redis-connect-mysql](connectors/mysql#redis-connect-mysql)
  * [Using redis-connect-oracle](connectors/oracle#redis-connect-oracle)
  * [Using redis-connect-postgres](connectors/postgres#redis-connect-postgres)
  * [Using redis-connect-sqlserver](connectors/mssql#redis-connect-sqlserver)
  * [Using redis-connect-files](#connectors/files#redis-connect-files)

## Technical Overview

Redis Connect is a distributed platform that enables near real-time replication and transformation (data pipelines) of row-level change-data events (Create, Update, Delete operations) from heterogeneous platforms to Redis Enterprise databases/modules. Redis Connect Jobs migrate source-database operations in the same time-order they were committed using event-driven workflows. It has a modular, extendable, and configurable architecture which provides the flexibility to deploy in a variety of topologies and cover multiple use-cases.

Redis Connect can also perform the function of event-sourcing, acting as both the message broker and event-store, so time-ordered history of change-data events are captured for auditing and/or replay purposes. In the occurrence of target-database downtime, change-data events will not be lost but instead resume replication, upon recovery, from the last committed checkpoint.

Redis Connect has a cloud-native shared-nothing architecture which allows any cluster node (Redis Connect Instance) to perform either/both Job Management and Job Execution functions. It is implemented and compiled in JAVA, which deploys on a platform-independent JVM, allowing Redis Connect instances to be agnostic of the underlying operating system (Linux, Windows, Docker Containers, etc.) Its lightweight design and minimal use of infrastructure-resources avoids complex dependencies on other distributed platforms such as Kafka and ZooKeeper. In fact, most uses of Redis Connect will only require the deployment of a few JVMs to handle Job Execution with high-availability.

Integration with source and target databases is handled by an extendable connector framework. Each Redis Connect Connector uniquely interfaces with its source database’s supported and built-in change data capture process. While there are multiple patterns to implement change data capture (polling-publisher, dual-writes, transaction-log tailing, etc.), Redis Connect connectors prioritize integration with source-database transaction-logs (when supported) to avoid impacting performance and availability.

## Use Cases

For one-time migrations, we commonly see customers attempting a blue-green deployment, where for some period of time, both the new and old deployment are available in parallel. Once the customer is confident with their new system, they have a hard cutoff for the legacy deployment. Redis Connect is a good fit here, since it can maintain consistency across both blue and green deployments until eventually cutting off replication. Now in the case of multi-phased migrations, such as a [strangler application](https://martinfowler.com/bliki/StranglerFigApplication.html) where we are breaking microservices off a monolith legacy database, Redis Connect can maintain consistency between heterogenous databases, for years, until the monolith is eventually retired for good. 

We’re also seeing a lot of demand within the analytics space, with customers that want to perform intra-day real-time ETL. In this case, Redis Connect is used more as an initial loader, which requires it to partition its consumption of source-side transactional data, so it can finish its ETL process within a thin SLA window. This is where another nice feature of Redis Connect is a value-add. For initial load jobs, Redis Connect can spawn child processes, which can in parallel consume partitioned data from the source. The partitioning strategy can be configured to fit the SLA window for the ETL job.

Another interesting use-case within analytics is building denormalized views for real-time reporting, alerts, and visualization and in this case Redis Connect can continuously replicate changed-data events from a relational database to a Redis Hash or String or JSON model. Since relational databases rely on SQL, its not a surprise that users for this use-case predominantly rely on [RediSearch](https://redislabs.com/modules/redis-search/) for real-time SQL-like querying based on secondary indexes.
