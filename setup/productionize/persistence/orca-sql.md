---
layout: single
title:  "Set up Orca to use SQL"
sidebar:
  nav: setup
---

{% include toc %}

Orca's execution state is stored in Redis by default, but can be configured for SQL. 
In this topology, Redis is still required for the work queue.
Using SQL for execution state will make your Spinnaker installation more durable.

This guide will go over MySQL setup, how to configure Orca, as well as how to perform a zero-downtime migration from Redis to SQL.

If you already have an Orca deployment, you should also refer to the [Redis to SQL Migration Guide](/guides/operator/orca-redis-to-sql/).

## MySQL 5.7 Setup

Orca ships with MySQL drivers by default, but you can include your own JDBC drivers on the classpath if you need to connect to a different database.

Orca has been developed and tested targeting MySQL 5.7. As part of this, setting MySQL's `tx_isolation` value to `READ-COMMITTED` is essential to successfully running Orca in SQL. 
While Orca will attempt to set this on connection sessions, it is better to have it set on the database itself.

The SQL integration is configured to support a `migration` user and a `service` user.
The `migration` user will only be used to perform schema changes on the database, whereas the `service` user will be used for runtime traffic.

Before deploying Orca, the schema and database uses must first be manually setup:

1. Set MySQL Server `tx_isolation` setting to `READ-COMMITTED`
2. Setup the schema and database users
  
  ```sql
  CREATE SCHEMA `orca` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

  GRANT 
    SELECT, INSERT, UPDATE, DELETE, EXECUTE, SHOW VIEW 
  ON `orca`.* 
  TO 'orca_service'@'%';

  GRANT 
    SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, REFERENCES, INDEX, ALTER, LOCK TABLES, EXECUTE, SHOW VIEW 
  ON `orca`.* 
  TO 'orca_migrate'@'%';
  ```

When Orca starts up, it will perform database migrations to ensure its running the correct schema.
It is safe to start multiple Orca services at the same time, even if migrations need to be run.

## Configuring Orca for SQL

This configuration is your baseline for Orca to talk to SQL, in orca.yml:

```yaml
sql:
  enabled: true
  connectionPool:
    jdbcUrl: jdbc:mysql://localhost:3306/orca
    user: orca_service
    password: hunter2
    connectionTimeout: 5000
    maxLifetime: 30000
    # MariaDB-specific:
    maxPoolSize: 50
  migration:
    jdbcUrl: jdbc:mysql://localhost:3306/orca
    user: orca_migrate
    password: hunter2

# Ensure we're only using SQL for accessing execution state
executionRepository:
  sql:
    enabled: true
  redis:
    enabled: false

# Reporting on active execution metrics will be handled by SQL
monitor:
  activeExecutions:
    redis: false
```

## Netflix's Amazon Aurora Example

While vanilla MySQL provides more durability and performance over Redis, Netflix additionally uses Amazon Aurora MySQL 5.7.
If you'd like to configure Orca to use Aurora as well, here is how Netflix has it set up.

**IMPORTANT**: This configuration is for multi-region Aurora replication.
If you are only deploying Aurora into a single region, don't enable any binlog settings.

#### Aurora Parameter Groups

- *binlog_cache_size*: `32768`
- *default_tmp_storage_engine*: `InnoDB`
- *general_log*: `0`
- *innodb_adaptive_hash_index*: `0`
- *innodb_buffer_pool_size*: `{DBInstanceClassMemory*3/4}`
- *key_buffer_size*: `16777216`
- *log_queries_not_using_indexes*: `0`
- *log_throttle_queries_not_using_indexes*: `60`
- *long_query_time*: `0.5`
- *max_allowed_packet*: `25165824`
- *max_binlog_size*: `134217728`
- *query_cache_size*: `{DBInstanceClassMemory/24}`
- *query_cache_type*: `1`
- *read_buffer_size*: `262144`
- *slow_query_log*: `1`
- *sync_binlog*: `1`
- *tx_isolation*: `READ-COMMITTED`

#### Aurora DB Parameters Group

- *binlog_checksum*: `NONE`
- *binlog_error_action*: `IGNORE_ERROR`
- *binlog_format*: `MIXED`
- *character_set_client*: `utf8mb4`
- *character_set_connection*: `utf8mb4`
- *character_set_database*: `utf8mb4`
- *character_set_filesystem*: `utf8mb4`
- *character_set_results*: `utf8mb4`
- *character_set_server*: `utf8mb4`
- *collation_connection*: `utf8mb4_unicode_ci`
- *collation_server*: `utf8mb4_unicode_ci`
- *innodb_checksums*: `0`

#### MariaDB

When using Aurora MySQL 5.7, you should also setup Orca to use the MariaDB JDBC driver over MySQL Connector.
The MariaDB driver is Aurora clustering aware, which takes care of automatic master failover operations. 
Due to licensing issues, Orca cannot ship with the MariaDB driver.

An example of wiring up MariaDB into Orca can be found here: [robzienert/orca-mariadb-extension](https://github.com/robzienert/orca-mariadb-extension).