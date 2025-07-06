# MySQL to Kafka CDC Implementation Guide

This repository contains a complete implementation guide for MySQL to Kafka Change Data Capture (CDC) using Debezium. The setup demonstrates real-time database change streaming with production-ready configurations.

## Overview

Change Data Capture enables real-time streaming of database changes to downstream systems. This implementation captures MySQL row-level changes (INSERT, UPDATE, DELETE) and streams them to Kafka topics with sub-100ms latency.

**Key Components:**

- MySQL with binary logging enabled
- Apache Kafka with KRaft mode (ZooKeeper-free)
- Debezium MySQL connector via Kafka Connect
- Two connector configurations (simplified and full event formats)

## Prerequisites

- Docker and Docker Compose installed
- Minimum 4GB RAM allocated to Docker
- Available ports: 3306 (MySQL), 8083 (Kafka Connect), 9092, 19092 (Kafka)
- MySQL client for database operations

## Installation and Setup

### 1. Start Infrastructure Services

```bash
# Start all services
docker-compose up -d

# Monitor startup progress
docker-compose logs -f connect

# Verify all containers are running
docker-compose ps
```

Wait approximately 2-3 minutes for all services to initialize. The Kafka Connect service takes the longest as it downloads and installs the Debezium connector.

### 2. Verify Service Readiness

```bash
# Check Kafka Connect API
curl http://localhost:8083/connector-plugins

# Verify Debezium MySQL connector is available
curl http://localhost:8083/connector-plugins | grep -i mysql
```

### 3. Configure MySQL Database

Connect to MySQL and create the required database objects:

```bash
mysql -h localhost -u root -prootpassword
```

Execute the following SQL commands:

```sql
USE testdb;

-- Create test table
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create CDC user with required permissions
CREATE USER 'debezium'@'%' IDENTIFIED BY 'dbz_password';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';
GRANT SELECT ON testdb.* TO 'debezium'@'%';
FLUSH PRIVILEGES;

-- Verify binary logging is enabled
SHOW VARIABLES LIKE 'log_bin';

EXIT;
```

## Connector Deployment

### Simple CDC Connector

Deploys a connector with unwrapped event format for simplified downstream consumption:

```bash
# Deploy connector
curl -X POST -H "Content-Type: application/json" \
  --data @mysql-connector.json \
  http://localhost:8083/connectors

# Verify deployment
curl http://localhost:8083/connectors/mysql-source-connector/status
```

The connector should show status "RUNNING" for both connector and task components.

### Full CDC Connector

Deploys a connector that preserves complete CDC event metadata:

```bash
# Deploy full format connector
curl -X POST -H "Content-Type: application/json" \
  --data @mysql-connector-full.json \
  http://localhost:8083/connectors

# Verify deployment
curl http://localhost:8083/connectors/mysql-full-connector/status
```

## Testing and Verification

### View Created Topics

```bash
# List Kafka topics
docker exec -it kafka-kraft kafka-topics --list --bootstrap-server kafka:29092

# Expected topics:
# mysql_server.testdb.users     (simplified format)
# mysql_full.testdb.users       (full format)
# Schema and internal topics
```

### Real-Time Event Monitoring

Set up consumers in separate terminal sessions to monitor events:

**Terminal 1 - Simplified Events:**

```bash
docker exec -it kafka-kraft kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic mysql_server.testdb.users \
  --from-beginning
```

**Terminal 2 - Full CDC Events:**

```bash
docker exec -it kafka-kraft kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic mysql_full.testdb.users \
  --from-beginning
```

### Database Operations Testing

Execute these operations and observe corresponding Kafka events:

```sql
-- INSERT operations
INSERT INTO users (name, email) VALUES ('John Doe', 'john@example.com');
INSERT INTO users (name, email) VALUES ('Jane Smith', 'jane@example.com');

-- UPDATE operations
UPDATE users SET email = 'john.doe@newdomain.com' WHERE id = 1;

-- DELETE operations
DELETE FROM users WHERE id = 2;
```

## Event Format Analysis

### Simplified Format Output

The unwrapped format produces clean JSON suitable for direct application consumption:

```json
// INSERT event
{"id":1,"name":"John Doe","email":"john@example.com","created_at":"2025-07-06T11:20:23Z"}

// UPDATE event
{"id":1,"name":"John Doe","email":"john.doe@newdomain.com","created_at":"2025-07-06T11:20:23Z"}

// DELETE event
null
```

### Full CDC Format Output

Complete CDC events with operation metadata and change context:

```json
// INSERT event
{
  "before": null,
  "after": {"id":1,"name":"John Doe","email":"john@example.com","created_at":"2025-07-06T11:20:23Z"},
  "source": {
    "version":"2.4.0.Final",
    "connector":"mysql",
    "name":"mysql_full",
    "ts_ms":1751801290000,
    "db":"testdb",
    "table":"users"
  },
  "op": "c",
  "ts_ms": 1751801290047
}

// UPDATE event
{
  "before": {"id":1,"name":"John Doe","email":"john@example.com","created_at":"2025-07-06T11:20:23Z"},
  "after": {"id":1,"name":"John Doe","email":"john.doe@newdomain.com","created_at":"2025-07-06T11:20:23Z"},
  "op": "u",
  "ts_ms": 1751801319186
}

// DELETE event
{
  "before": {"id":2,"name":"Jane Smith","email":"jane@example.com","created_at":"2025-07-06T11:21:17Z"},
  "after": null,
  "op": "d",
  "ts_ms": 1751801400000
}
```

**Operation Types:**

- `c` = CREATE (INSERT)
- `u` = UPDATE
- `d` = DELETE

## Operational Commands

### Connector Management

```bash
# List all connectors
curl http://localhost:8083/connectors

# Get connector configuration
curl http://localhost:8083/connectors/mysql-source-connector/config

# Check detailed connector status
curl http://localhost:8083/connectors/mysql-source-connector/status

# Pause connector
curl -X PUT http://localhost:8083/connectors/mysql-source-connector/pause

# Resume connector
curl -X PUT http://localhost:8083/connectors/mysql-source-connector/resume

# Delete connector
curl -X DELETE http://localhost:8083/connectors/mysql-source-connector

# Restart connector
curl -X POST http://localhost:8083/connectors/mysql-source-connector/restart

# Restart specific task
curl -X POST http://localhost:8083/connectors/mysql-source-connector/tasks/0/restart
```

### Topic Management

```bash
# List all topics
docker exec -it kafka-kraft kafka-topics --list --bootstrap-server kafka:29092

# Describe topic configuration
docker exec -it kafka-kraft kafka-topics --describe \
  --topic mysql_server.testdb.users --bootstrap-server kafka:29092

# Create additional topic
docker exec -it kafka-kraft kafka-topics --create \
  --topic custom-topic --partitions 3 --replication-factor 1 \
  --bootstrap-server kafka:29092

# Delete topic
docker exec -it kafka-kraft kafka-topics --delete \
  --topic mysql_server.testdb.users --bootstrap-server kafka:29092
```

### Consumer Operations

```bash
# Consume from beginning
docker exec -it kafka-kraft kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic mysql_server.testdb.users \
  --from-beginning

# Consume latest messages only
docker exec -it kafka-kraft kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic mysql_server.testdb.users

# Consume with key and timestamp information
docker exec -it kafka-kraft kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic mysql_server.testdb.users \
  --property print.key=true \
  --property print.timestamp=true \
  --from-beginning

# Consume specific number of messages
docker exec -it kafka-kraft kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic mysql_server.testdb.users \
  --max-messages 10 \
  --from-beginning
```

## Advanced Testing Scenarios

### Schema Evolution Testing

Test how CDC handles database schema changes:

```sql
-- Add new column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Insert data with new schema
INSERT INTO users (name, email, phone) VALUES ('Bob Wilson', 'bob@example.com', '555-0123');

-- Modify column
ALTER TABLE users MODIFY COLUMN phone VARCHAR(30);

-- Drop column
ALTER TABLE users DROP COLUMN phone;
```

### Bulk Operations Testing

Test CDC performance with large data changes:

```sql
-- Bulk insert test
INSERT INTO users (name, email)
SELECT
  CONCAT('User', n) as name,
  CONCAT('user', n, '@example.com') as email
FROM (
  SELECT a.N + b.N * 10 + 1 n
  FROM
  (SELECT 0 AS N UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) a,
  (SELECT 0 AS N UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) b
) numbers;

-- Bulk update test
UPDATE users SET email = CONCAT('updated_', email) WHERE id > 10;

-- Bulk delete test
DELETE FROM users WHERE id > 50;
```

### Transaction Testing

Verify CDC handles database transactions correctly:

```sql
-- Test committed transaction
BEGIN;
INSERT INTO users (name, email) VALUES ('Transaction User 1', 'trans1@example.com');
UPDATE users SET email = 'trans1_updated@example.com' WHERE name = 'Transaction User 1';
COMMIT;

-- Test rolled back transaction (should not appear in CDC)
BEGIN;
INSERT INTO users (name, email) VALUES ('Rollback User', 'rollback@example.com');
ROLLBACK;
```

## Monitoring and Debugging

### Service Health Checks

```bash
# Check container status
docker-compose ps

# Monitor resource usage
docker stats

# View service logs
docker logs kafka-connect
docker logs mysql-cdc
docker logs kafka-kraft
```

### MySQL Monitoring

```bash
# Check binary log status
mysql -h localhost -u root -prootpassword -e "SHOW MASTER STATUS;"

# View binary log files
mysql -h localhost -u root -prootpassword -e "SHOW BINARY LOGS;"

# Check replication variables
mysql -h localhost -u root -prootpassword -e "SHOW VARIABLES LIKE '%binlog%';"

# Monitor active connections
mysql -h localhost -u root -prootpassword -e "SHOW PROCESSLIST;"
```

### Kafka Cluster Health

```bash
# Check broker information
docker exec -it kafka-kraft kafka-broker-api-versions --bootstrap-server kafka:29092

# View consumer groups
docker exec -it kafka-kraft kafka-consumer-groups --bootstrap-server kafka:29092 --list

# Check consumer lag
docker exec -it kafka-kraft kafka-consumer-groups --bootstrap-server kafka:29092 \
  --group connect-mysql-source-connector --describe
```

## Troubleshooting Guide

### Common Issues and Solutions

**Connector Fails to Start**

- Verify MySQL user permissions are correctly granted
- Check that binary logging is enabled: `SHOW VARIABLES LIKE 'log_bin';`
- Ensure MySQL is accessible from the connector container
- Review connector logs: `docker logs kafka-connect`

**No Events in Kafka Topics**

- Confirm connector status shows "RUNNING": `curl http://localhost:8083/connectors/mysql-source-connector/status`
- Verify topics were created: `docker exec -it kafka-kraft kafka-topics --list --bootstrap-server kafka:29092`
- Check for connector errors in logs
- Ensure database operations are being performed on monitored tables

**Consumer Cannot Read Messages**

- Verify correct topic name and bootstrap servers
- Check if consumer is reading from the beginning: `--from-beginning`
- Confirm Kafka cluster is healthy
- Test with simple console consumer first

**High Latency or Missing Events**

- Check MySQL binary log retention settings
- Monitor connector task lag and processing rates
- Verify network connectivity between services
- Review connector configuration for performance settings

### Performance Optimization

**MySQL Configuration**

```ini
# Optimize for CDC workload
innodb_buffer_pool_size = 1G
innodb_log_file_size = 256M
max_binlog_size = 100M
binlog_expire_logs_seconds = 86400
sync_binlog = 1
```

**Connector Tuning**

```json
{
  "max.batch.size": "2048",
  "max.queue.size": "8192",
  "poll.interval.ms": "500",
  "snapshot.max.threads": "2"
}
```

## Configuration Reference

### Files in This Repository

- **`docker-compose.yml`** - Infrastructure setup with MySQL, Kafka, and Kafka Connect
- **`mysql-connector.json`** - Simplified CDC connector configuration
- **`mysql-connector-full.json`** - Full CDC event format connector configuration

### Key Configuration Parameters

**MySQL Connector Settings:**

- `database.server.name` - Logical server name used as topic prefix
- `snapshot.mode` - Initial snapshot behavior (initial, schema_only, never)
- `transforms` - Event transformation pipeline
- `table.include.list` - Specific tables to monitor
- `schema.history.internal.kafka.topic` - Schema change tracking topic

**Performance Settings:**

- `max.batch.size` - Events per batch to Kafka
- `max.queue.size` - Internal connector queue size
- `poll.interval.ms` - Database polling frequency

## Cleanup and Maintenance

### Stop Services

```bash
# Stop all services
docker-compose down

# Stop and remove volumes (deletes all data)
docker-compose down -v
```

### Reset Environment

```bash
# Remove all containers and volumes
docker-compose down -v

# Remove images to free disk space
docker system prune -a

# Restart clean environment
docker-compose up -d
```

## Production Considerations

### Security

- Use dedicated database users with minimal required permissions
- Enable TLS encryption for all connections
- Implement proper network isolation
- Secure Kafka Connect REST API

### Monitoring

- Monitor connector health and task status
- Track replication lag and event processing rates
- Set up alerts for connector failures
- Monitor database binary log retention

### Scalability

- Consider multiple connectors for large schemas
- Implement proper partitioning strategies
- Plan for schema evolution and compatibility
- Monitor resource utilization and scale accordingly

This implementation provides a solid foundation for understanding and implementing MySQL to Kafka CDC in production environments.
