# redis-connect-mysql

redis-connect-mysql is a Redis Connect connector for capturing changes (INSERT, UPDATE and DELETE) from MySQL (source) and writing them to a Redis Enterprise database (Target). redis-connect-mysql cdc connector implementation is based on <a href="https://debezium.io/documentation/reference/stable/connectors/mysql.html" target="_blank">Debezium</a>, which is an open source distributed platform for change data capture.

The first time redis-connect-mysql connects to a MySQL database, it reads a consistent snapshot of all the schemas.
When that snapshot is complete, the connector continuously streams the changes that were committed to MySQL and generates a corresponding insert, update or delete event.
All the events for each table(s) are recorded in a separate [Redis data structure or module](../../docs/writers.md) of your choice, where they can be easily consumed by applications and services.

## Architecture

![Redis Connect high-level Architecture](/docs/images/RedisConnect_Arch.png)
<b>Redis Connect high-level Architecture Diagram</b>

Redis Connect has a cloud-native shared-nothing architecture which allows any cluster node (Redis Connect Instance) to perform either/both Job Management and Job Execution functions. It is implemented and compiled in JAVA, which deploys on a platform-independent JVM, allowing Redis Connect instances to be agnostic of the underlying operating system (Linux, Windows, Docker Containers, etc.) Its lightweight design and minimal use of infrastructure-resources avoids complex dependencies on other distributed platforms such as Kafka and ZooKeeper. In fact, most uses of Redis Connect will only require the deployment of a few JVMs to handle Job Execution and Job Management with high-availability.

<p>
On their own Redis Connect instances are stateless therefore require Redis to manage Job Management and Job Execution state - such as checkpoints, claims, optional intermediary data storage, etc. With this design, Redis Connect instances can fail/fail-over without risking data loss, duplication, and/or order. As long as another Redis Connect instance is actively available to claim responsibility for Job Execution, or can be recovered, it will pick up from the last recorded checkpoint.

<h5>Redis Connect Components</h5>

<h6>Redis Connect Instance</h6>
<p>A Redis Connect instance is a single JVM that executes one or more pipelines.

<h6>Pipeline</h6>
<p>A Pipeline moves, transforms and orchestrates data transfer from one data structure in source data store to another data structure in target data source.

<h6>Job</h6>
<p>A Job is an implementation of a pipeline. One pipeline can have only one implementation. A job is considered “assigned” if a Redis Connect instance is executing the job. Redis Connect instance executes one or more Job processes that
<br>• Read data, in batch, from the data structure on the source data store
<br>• Transforms and maps data to a predefined data structure on the target data store
<br>• Writes data, in batch, to the data structure on the target data store

<h6>Job Manager</h6>
<p>Job Manager is a wrapper process that instantiates Job Reaper and Job Claimer processes.

<h6>Job Reaper</h6>
<p>Job Reaper is a process, within a Redis Connect instance, that tracks the status of all jobs. If any Jobs are not being executed, then the reaper process makes them available to be “assigned”. A single job reaper process is instantiated within each Redis Connect instance. Only one job reaper process is active across all Redis Connect instances.

<h6>Job Claimer</h6>
<p>Job Claimer is a process, within a Redis Connect instance that initiates <i>UNASSIGNED</i> jobs. A single job claimer process is instantiated within each Redis Connect instance. All job claimer processes are active across all Redis Connect instances.

## Setting up MySQL (Source)

Please see <a href="https://debezium.io/documentation/reference/stable/connectors/mysql.html#setting-up-mysql" target="_blank">MySQL Setup</a> for reference.

Please see an example under [Demo](demo/setup_mysql.sh).

## Setting up Redis Enterprise Databases (Target)

Before using the MySQL connector (redis-connect-mysql) to capture the changes committed on MySQL into Redis Enterprise Database, first create a database for the metadata management and metrics provided by Redis Connect by creating a database with [RedisTimeSeries](https://redis.com/modules/redis-timeseries/) module enabled, see [Create Redis Enterprise Database](https://docs.redis.com/latest/rs/administering/creating-databases/#creating-a-new-redis-database) for reference. Then, create (or use an existing) another Redis Enterprise database (Target) to store the changes coming from MySQL. Additionally, you can enable [RediSearch 2.0](https://redis.com/blog/introducing-redisearch-2-0/) module on the target database to enable secondary index with full-text search capabilities on the existing hashes where MySQL changed events are being written at then [create an index, and start querying](https://oss.redis.com/redisearch/Commands/) the document in hashes.

## Download and Setup

---

### Minimum Hardware Requirements

* 1GB of RAM
* 4 CPU cores
* 20GB of disk space
* 1G Network
* JRE 8+ (JRE 11 is preferred)

**NOTE**

The current [release](https://github.com/RedisLabs-Field-Engineering/redis-connect-dist/releases) has been built with JDK 11 and tested with JRE 11 and above. Please have JRE 11+ installed prior to running this connector.

---

Download the [latest release](https://github.com/RedisLabs-Field-Engineering/redis-connect-dist/releases) and un-tar redis-connect-mysql-`<version>.<build>`.tar.gz archive.

All the contents would be extracted under redis-connect-mysql

Contents of redis-connect-mysql
<br>• bin – contains script files
<br>• lib – contains java libraries
<br>• config – contains sample config files for cdc and initial loader jobs
<br>• extlib – directory to copy [custom stage](https://github.com/RedisLabs-Field-Engineering/redis-connect-custom-stage-demo) implementation jar(s)

## Redis Connect Setup and Job Management Configurations

Copy the _sample_ directory and it's contents i.e. _yml_ files, _mappers_ and templates folder under _config_ directory to the name of your choice e.g. `redis-connect-mysql$ cp -R config/samples/mysql config/<project_name>` or reuse sample folder as is and edit/update the configuration values according to your setup.

#### Configuration files

<details><summary>Configure logback.xml</summary>
<p>

#### logging configuration file.

### Sample logback.xml under redis-connect-mysql/config folder

```xml
<configuration debug="true" scan="true" scanPeriod="15 seconds">

    <property name="START_UP_PATH" value="logs/redis-connect-startup.log"/>
    <property name="LOG_PATH" value="logs/redis-connect.log"/>

    <appender name="STARTUP" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${START_UP_PATH}</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/archived/startup.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <!-- each archived file, size max 10MB -->
            <maxFileSize>10MB</maxFileSize>
            <!-- total size of all archive files, if total size > 20GB, it will delete old archived file -->
            <totalSizeCap>20GB</totalSizeCap>
            <!-- 60 days to keep -->
            <maxHistory>60</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d %p %c{1.} [%t] %m%n</pattern>
        </encoder>
    </appender>

    <appender name="REDISCONNECT" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/archived/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <!-- each archived file, size max 10MB -->
            <maxFileSize>10MB</maxFileSize>
            <!-- total size of all archive files, if total size > 20GB, it will delete old archived file -->
            <totalSizeCap>20GB</totalSizeCap>
            <!-- 60 days to keep -->
            <maxHistory>60</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d %p %c{1.} [%t] %m%n</pattern>
        </encoder>
    </appender>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="startup" level="INFO" additivity="false">
        <appender-ref ref="STARTUP"/>
        <appender-ref ref="CONSOLE" />
    </logger>

    <logger name="redisconnect" level="INFO" additivity="false">
        <appender-ref ref="REDISCONNECT"/>
        <appender-ref ref="CONSOLE" />
    </logger>


    <logger name="com.redislabs" level="INFO" additivity="false">
        <appender-ref ref="REDISCONNECT"/>
        <appender-ref ref="CONSOLE" />
    </logger>
    <logger name="io.netty" level="OFF" additivity="false">
        <appender-ref ref="REDISCONNECT"/>
        <appender-ref ref="CONSOLE" />
    </logger>
    <logger name="io.lettuce" level="OFF" additivity="false">
        <appender-ref ref="REDISCONNECT"/>
        <appender-ref ref="CONSOLE" />
    </logger>
    <logger name="com.zaxxer" level="OFF" additivity="false">
        <appender-ref ref="REDISCONNECT"/>
        <appender-ref ref="CONSOLE"/>
    </logger>
    <logger name="io.debezium" level="INFO" additivity="false">
        <appender-ref ref="REDISCONNECT"/>
        <appender-ref ref="CONSOLE"/>
    </logger>
    <logger name="org.apache.kafka" level="OFF" additivity="false">
        <appender-ref ref="REDISCONNECT"/>
        <appender-ref ref="CONSOLE"/>
    </logger>
    <logger name="org.springframework" level="OFF" additivity="false">
        <appender-ref ref="REDISCONNECT"/>
        <appender-ref ref="CONSOLE"/>
    </logger>

    <root>
        <appender-ref ref="STARTUP"/>
        <appender-ref ref="REDISCONNECT"/>
    </root>

</configuration>
```

</p>
</details>

<details><summary>Configure env.yml</summary>
<p>

#### Environment configuration file with source and target connection information.

Redis URI syntax is described [here](https://github.com/lettuce-io/lettuce-core/wiki/Redis-URI-and-connection-details#uri-syntax).

### Sample env.yml under redis-connect-mysql/config/samples/[mysql|loader] folder. Any of these fields (values) can be replaced by environment variables.

```yml
connections:
  - id: jobConfigConnection
    type: Redis
    url: redis://${REDISCONNECT_TARGET_USERNAME}:${REDISCONNECT_TARGET_PASSWORD}@127.0.0.1:14001
  - id: targetConnection
    type: Redis
    url: redis://${REDISCONNECT_TARGET_USERNAME}:${REDISCONNECT_TARGET_PASSWORD}@127.0.0.1:14000
  - id: metricsConnection
    type: Redis
    url: redis://${REDISCONNECT_TARGET_USERNAME}:${REDISCONNECT_TARGET_PASSWORD}@127.0.0.1:14001
  - id: RDBConnection
    type: RDB
    name: RedisConnect #database pool name
    database: RedisConnect #database
    url: "jdbc:mysql://127.0.0.1:3306/RedisConnect?useInformationSchema=true"
    host: 127.0.0.1
    port: 3306
    username: ${REDISCONNECT_SOURCE_USERNAME}
    password: ${REDISCONNECT_SOURCE_PASSWORD}
```

</p>
</details>

<details><summary>Configure Setup.yml</summary>
<p>

#### Environment level configurations.

### Sample Setup.yml under redis-connect-mysql/config/samples/mysql folder

```yml
connectionId: jobConfigConnection
job:
  metrics:
    connectionId: metricsConnection
    retentionInHours: 12
    keys:
      - key: "RedisConnect:emp:C:Throughput"
        retentionInHours: 4
        labels:
          schema: RedisConnect
          table: emp
          op: C
      - key: "RedisConnect:emp:U:Throughput"
        retentionInHours: 4
        labels:
          schema: RedisConnect
          table: emp
          op: U
      - key: "RedisConnect:emp:D:Throughput"
        retentionInHours: 4
        labels:
          schema: RedisConnect
          table: emp
          op: D
      - key: "RedisConnect:emp:Latency"
        retentionInHours: 4
        labels:
          schema: RedisConnect
          table: emp
  jobConfig:
    - name: RedisConnect-mysql
      config: JobConfig.yml
      variables:
        database: RedisConnect
        sourceValueTranslator: SOURCE_RECORD_TO_OP_TRANSLATOR
```

</p>
</details>

<details><summary>Configure JobManager.yml</summary>
<p>

#### Configuration for Job Reaper and Job Claimer processes.

### Sample JobManager.yml under redis-connect-mysql/config/samples/mysql folder

```yml
connectionId: jobConfigConnection
metricsReporter:
  - REDIS_TS_METRICS_REPORTER
```

</p>
</details>

<details><summary>Configure JobConfig.yml</summary>
<p>

#### Job level details. Please see [writers](../../docs/writers.md) for other write stage usages.

### Sample JobConfig.yml under redis-connect-mysql/config/samples/mysql folder

You can have one or more JobConfig.yml (or with any name e.g. JobConfig-<table_name>.yml) and specify them in the Setup.yml under jobConfig: tag. If specifying more than one table (as below) then make sure maxNumberOfJobs: tag under JobManager.yml is set accordingly e.g. if maxNumberOfJobs: tag is set to 2 then Redis Connect will start 2 cdc jobs under the same JVM instance. If the workload is more and you want to spread out (scale) the cdc jobs then create multiple JobConfig's and specify them in the Setup.yml under jobConfig: tag.

```yml
jobId: ${jobId}
producerConfig:
  producerId: RDB_EVENT_PRODUCER
  connectionId: RDBConnection
  tables:
    - RedisConnect.emp #schema.table
  metricsEnabled: false
pipelineConfig:
  eventTranslator: "${sourceValueTranslator}"
  checkpointConfig:
    providerId: RDB_SQL_CHECKPOINT_READER
    connectionId: targetConnection
    checkpoint: "${jobId}-${database}"
    rdbType: "mysql"
  stages:
    HashWriteStage:
      handlerId: REDIS_HASH_WRITER
      connectionId: targetConnection
      metricsEnabled: false
      prependTableNameToKeys: true
      deleteOnKeyUpdate: true
      async: true
    CheckpointStage:
      handlerId: REDIS_HASH_CHECKPOINT_WRITER
      connectionId: targetConnection
      metricEnabled: false
      async: true
      checkpoint: "${jobId}-${database}"
```

</p>
</details>

<details><summary>Configure mapper.yml</summary>
<p>

#### mapper configuration file.

### Sample mapper.yml under redis-connect-mysql/config/samples/mysql/mappers folder

```yml
schema: RedisConnect
tables:
  - table: emp
    mapper:
      id: Test
      processorID: Test
      publishBefore: false
      passThrough: false
      ignoreReads: false # false reloads the data from last checkpoint
      columns:
        - src: empno
          target: EmployeeNumber
          type: INT
          publishBefore: false
        - src: fname
          target: FirstName
        - src: lname
          target: LastName
        - src: job
          target: Job
        - src: mgr
          target: Manager
          type: INT
        - src: hiredate
          target: HireDate
          type: DATE_TIME
        - src: sal
          target: Salary
          type: DOUBLE
        - src: comm
          target: Commission
          type: DOUBLE
        - src: dept
          target: Department
          type: INT
```

If you don't need any transformation of source columns then you can simply use passThrough option and you don't need to explicitly map each source columns to Redis target data structure.

```yml
schema: RedisConnect # Schema name e.g. dbo. One mapper file per schema and you can have multiple tables in the same mapper file as long as schema is same, otherwise create multiple mapper files e.g. mapper1.xml, mapper2.xml or <table_name>.xml etc. under mappers' folder of your config dir.
tables:
  - table: emp # emp table under dbo schema
    mapper:
      id: Test
      processorID: Test
      publishBefore: false # publishBefore - Global setting, that specifies if before values have to be published for all columns. This setting could be overridden at each column level
      passThrough: true # set it to true if you don't need to map individual columns. You always need to have the key column mappings.
      columns:
        - src: empno # key column on the source emp table
          target: empno
          type: INT
          publishBefore: false
```

</p>
</details>

## Start Redis Connect MySQL Connector
<details><summary>Execute Redis Connect startup script to see all the options</summary>
<p>

```bash
redis-connect-mysql/bin$ ./redisconnect.sh    
-------------------------------
Redis Connect startup script.
*******************************
Please ensure that the value of REDISCONNECT_CONFIG points to the correct config directory in /home/viragtripathi/redis-connect-mysql/bin/redisconnect.conf before executing any of the options below
*******************************
Usage: [-h|cli|stage|start]
options:
-h: Print this help message and exit.
cli: starts redis-connect-cli.
stage: clean and stage redis database with cdc or initial loader job configurations.
start: start Redis Connect instance with provided cdc or initial loader job configurations.
-------------------------------
```

</p>
</details>

<h4>Stage Redis Connect Job</h4>
Before starting a Redis Connect instance, job config data needs to be seeded into Redis Config database from Job Configuration files. Configuration is provided in Setup.yml. After the configuration files are modified as needed, execute the startup script with <i>stage</i> option.

```bash
redis-connect-mysql/bin$ ./redisconnect.sh stage
```

<h4>Start Redis Connect Job</h4>
Once staging is done, execute the same script with <i>start</i> option to start the configured Job(s) i.e. an instance of Redis Connect.

```bash
redis-connect-mysql/bin$ ./redisconnect.sh start
```

| ℹ️                                         |
|:-------------------------------------------|
| Quick Start: Follow the [demo](demo)       |
| K8s Setup: Follow the [k8s-docs](k8s-docs) |
