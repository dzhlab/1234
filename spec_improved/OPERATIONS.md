# Operations Manual
## RG System - Production Operations Guide

> **Version:** 2.8
> **Date:** 2025-11-27
> **Status:** Production Ready
> **Audience:** DevOps, SRE, System Administrators

---

## Table of Contents

1. [Operations Overview](#operations-overview)
2. [System Architecture](#system-architecture)
3. [Monitoring & Alerting](#monitoring--alerting)
4. [Logging](#logging)
5. [Backup & Recovery](#backup--recovery)
6. [Performance Tuning](#performance-tuning)
7. [Troubleshooting](#troubleshooting)
8. [Runbooks](#runbooks)
9. [Maintenance Windows](#maintenance-windows)
10. [Disaster Recovery](#disaster-recovery)
11. [Capacity Planning](#capacity-planning)
12. [On-Call Procedures](#on-call-procedures)

---

## Operations Overview

### Service Level Objectives (SLOs)

| Metric | Target | Measurement Window |
|--------|--------|-------------------|
| **Availability** | 99.9% | Monthly |
| **API Response Time (p95)** | < 200ms | 5 minutes |
| **API Response Time (p99)** | < 500ms | 5 minutes |
| **WebSocket Latency** | < 100ms | 5 minutes |
| **Error Rate** | < 0.1% | 5 minutes |
| **Database Query Time (p95)** | < 50ms | 5 minutes |
| **Backup Success Rate** | 100% | Daily |
| **Recovery Time Objective (RTO)** | < 1 hour | Per incident |
| **Recovery Point Objective (RPO)** | < 15 minutes | Per incident |

### Error Budget

```
Monthly Error Budget = (1 - SLO) × Total Time
For 99.9% SLO: 0.1% × 30 days × 24 hours × 60 minutes = 43.2 minutes/month
```

**Error Budget Policy:**
- **100% budget remaining**: All releases allowed, normal velocity
- **50-100% budget remaining**: Proceed with caution, increase testing
- **25-50% budget remaining**: Focus on reliability, defer non-critical features
- **<25% budget remaining**: Feature freeze, focus only on reliability improvements

### Key Performance Indicators (KPIs)

```yaml
Service Health:
  - Uptime: 99.9%
  - Mean Time Between Failures (MTBF): > 720 hours (30 days)
  - Mean Time To Repair (MTTR): < 30 minutes

Performance:
  - Requests per second: 1000 RPS sustained
  - Concurrent users: 500 users
  - Database connections: < 80% pool utilization

Reliability:
  - Deployment success rate: > 95%
  - Rollback rate: < 5%
  - Failed health checks: < 1%
```

---

## System Architecture

### Production Environment

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                             │
└─────────────────┬───────────────────────────────────────────┘
                  │
         ┌────────▼─────────┐
         │   Cloudflare     │  ← DDoS Protection, CDN
         │   WAF + CDN      │
         └────────┬─────────┘
                  │
         ┌────────▼─────────┐
         │  Load Balancer   │  ← Nginx (2 instances)
         │  (HA Proxy)      │
         └────────┬─────────┘
                  │
      ┌───────────┴───────────┐
      │                       │
┌─────▼──────┐         ┌─────▼──────┐
│  API-1     │         │  API-2     │  ← Node.js API servers
│  (Primary) │         │  (Replica) │
└─────┬──────┘         └─────┬──────┘
      │                       │
      └───────────┬───────────┘
                  │
      ┌───────────┴───────────┐
      │                       │
┌─────▼──────┐         ┌─────▼──────┐
│PostgreSQL  │◄────────│PostgreSQL  │
│  Primary   │         │  Standby   │  ← Streaming replication
└─────┬──────┘         └────────────┘
      │
┌─────▼──────┐
│  Redis     │  ← Session store, cache
│  Sentinel  │
└────────────┘
```

### Infrastructure Components

| Component | Count | Specs | Purpose |
|-----------|-------|-------|---------|
| **Load Balancer** | 2 | 2 vCPU, 4GB RAM | HA Proxy + Nginx |
| **API Server** | 2 | 4 vCPU, 8GB RAM | Node.js application |
| **WebSocket Server** | 2 | 2 vCPU, 4GB RAM | Real-time communication |
| **PostgreSQL Primary** | 1 | 8 vCPU, 16GB RAM, 500GB SSD | Primary database |
| **PostgreSQL Standby** | 1 | 8 vCPU, 16GB RAM, 500GB SSD | Streaming replica |
| **Redis Primary** | 1 | 2 vCPU, 4GB RAM | Cache + sessions |
| **Redis Replica** | 1 | 2 vCPU, 4GB RAM | Read replica |
| **Monitoring** | 1 | 4 vCPU, 8GB RAM | Prometheus + Grafana |
| **Logging** | 1 | 4 vCPU, 16GB RAM, 1TB HDD | ELK Stack |

### Network Configuration

```yaml
VPC: 10.0.0.0/16

Subnets:
  Public:
    - 10.0.1.0/24  # Load Balancers (AZ-A)
    - 10.0.2.0/24  # Load Balancers (AZ-B)

  Private - Application:
    - 10.0.10.0/24 # API Servers (AZ-A)
    - 10.0.11.0/24 # API Servers (AZ-B)

  Private - Database:
    - 10.0.20.0/24 # PostgreSQL, Redis (AZ-A)
    - 10.0.21.0/24 # PostgreSQL, Redis (AZ-B)

  Private - Monitoring:
    - 10.0.30.0/24 # Prometheus, Grafana, ELK

Security Groups:
  lb-sg:
    - Inbound: 80/tcp, 443/tcp from 0.0.0.0/0
    - Outbound: 3000/tcp to api-sg

  api-sg:
    - Inbound: 3000/tcp from lb-sg
    - Outbound: 5432/tcp to db-sg, 6379/tcp to redis-sg

  db-sg:
    - Inbound: 5432/tcp from api-sg
    - Outbound: deny all

  redis-sg:
    - Inbound: 6379/tcp from api-sg
    - Outbound: deny all
```

---

## Monitoring & Alerting

### Metrics Collection

**Prometheus Configuration:**

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    environment: 'prod'

scrape_configs:
  # API Servers
  - job_name: 'api'
    static_configs:
      - targets:
          - '10.0.10.10:9090'  # API-1
          - '10.0.10.11:9090'  # API-2
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  # PostgreSQL
  - job_name: 'postgres'
    static_configs:
      - targets:
          - '10.0.20.10:9187'  # postgres_exporter

  # Redis
  - job_name: 'redis'
    static_configs:
      - targets:
          - '10.0.20.20:9121'  # redis_exporter

  # Node Exporter (system metrics)
  - job_name: 'node'
    static_configs:
      - targets:
          - '10.0.10.10:9100'
          - '10.0.10.11:9100'
          - '10.0.20.10:9100'
          - '10.0.20.20:9100'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'

rule_files:
  - '/etc/prometheus/rules/*.yml'
```

### Application Metrics

**Express.js Instrumentation:**

```typescript
import prometheus from 'prom-client';
import express from 'express';

// Create a Registry
const register = new prometheus.Registry();

// Add default metrics
prometheus.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_ms',
  help: 'Duration of HTTP requests in ms',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [10, 50, 100, 200, 500, 1000, 2000, 5000]
});

const httpRequestTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

const activeConnections = new prometheus.Gauge({
  name: 'websocket_active_connections',
  help: 'Number of active WebSocket connections'
});

const databaseQueryDuration = new prometheus.Histogram({
  name: 'database_query_duration_ms',
  help: 'Duration of database queries in ms',
  labelNames: ['query_type'],
  buckets: [5, 10, 25, 50, 100, 250, 500, 1000]
});

const cacheHitRate = new prometheus.Counter({
  name: 'cache_operations_total',
  help: 'Total cache operations',
  labelNames: ['operation', 'result'] // hit, miss
});

// Register custom metrics
register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestTotal);
register.registerMetric(activeConnections);
register.registerMetric(databaseQueryDuration);
register.registerMetric(cacheHitRate);

// Middleware to track HTTP metrics
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    const route = req.route?.path || req.path;

    httpRequestDuration
      .labels(req.method, route, res.statusCode.toString())
      .observe(duration);

    httpRequestTotal
      .labels(req.method, route, res.statusCode.toString())
      .inc();
  });

  next();
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### Alert Rules

```yaml
# /etc/prometheus/rules/alerts.yml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      # High Error Rate
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: critical
          component: api
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"

      # High Response Time
      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_ms_bucket[5m])) by (le)
          ) > 500
        for: 10m
        labels:
          severity: warning
          component: api
        annotations:
          summary: "High API response time"
          description: "95th percentile response time is {{ $value }}ms (threshold: 500ms)"

      # Service Down
      - alert: ServiceDown
        expr: up{job="api"} == 0
        for: 2m
        labels:
          severity: critical
          component: api
        annotations:
          summary: "API service is down"
          description: "API instance {{ $labels.instance }} is down"

  - name: database_alerts
    interval: 30s
    rules:
      # High Database Connections
      - alert: HighDatabaseConnections
        expr: |
          (
            pg_stat_database_numbackends
            /
            pg_settings_max_connections
          ) > 0.8
        for: 5m
        labels:
          severity: warning
          component: database
        annotations:
          summary: "High number of database connections"
          description: "Using {{ $value | humanizePercentage }} of max connections"

      # Replication Lag
      - alert: ReplicationLag
        expr: |
          (
            pg_replication_lag_seconds > 60
          )
        for: 5m
        labels:
          severity: warning
          component: database
        annotations:
          summary: "PostgreSQL replication lag is high"
          description: "Replication lag is {{ $value }}s (threshold: 60s)"

      # Disk Space Low
      - alert: DatabaseDiskSpaceLow
        expr: |
          (
            node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"}
            /
            node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}
          ) < 0.15
        for: 10m
        labels:
          severity: critical
          component: database
        annotations:
          summary: "Database disk space is low"
          description: "Only {{ $value | humanizePercentage }} disk space remaining"

  - name: redis_alerts
    interval: 30s
    rules:
      # Redis Down
      - alert: RedisDown
        expr: redis_up == 0
        for: 2m
        labels:
          severity: critical
          component: cache
        annotations:
          summary: "Redis is down"
          description: "Redis instance {{ $labels.instance }} is not responding"

      # High Memory Usage
      - alert: RedisHighMemory
        expr: |
          (
            redis_memory_used_bytes
            /
            redis_memory_max_bytes
          ) > 0.9
        for: 5m
        labels:
          severity: warning
          component: cache
        annotations:
          summary: "Redis memory usage is high"
          description: "Redis is using {{ $value | humanizePercentage }} of max memory"
```

### Grafana Dashboards

**API Performance Dashboard:**

```json
{
  "dashboard": {
    "title": "RG System - API Performance",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (method)"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Response Time (p95, p99)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_ms_bucket[5m])) by (le))",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_ms_bucket[5m])) by (le))",
            "legendFormat": "p99"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status_code=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))"
          }
        ],
        "type": "singlestat",
        "format": "percentunit"
      },
      {
        "title": "Active WebSocket Connections",
        "targets": [
          {
            "expr": "websocket_active_connections"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

### AlertManager Configuration

```yaml
# /etc/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

route:
  group_by: ['alertname', 'component']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    # Critical alerts go to PagerDuty + Slack
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true

    - match:
        severity: critical
      receiver: 'slack-critical'

    # Warning alerts go to Slack only
    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#rgsystem-alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'slack-critical'
    slack_configs:
      - channel: '#rgsystem-critical'
        title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        color: 'danger'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#rgsystem-warnings'
        title: '⚠️  WARNING: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        color: 'warning'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}: {{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

inhibit_rules:
  # Inhibit warning if critical alert is firing
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'component']
```

---

## Logging

### Centralized Logging (ELK Stack)

**Filebeat Configuration:**

```yaml
# /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/rgsystem/api/*.log
    fields:
      app: rgsystem-api
      environment: production
    json.keys_under_root: true
    json.add_error_key: true

  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      app: nginx
      log_type: access

  - type: log
    enabled: true
    paths:
      - /var/log/nginx/error.log
    fields:
      app: nginx
      log_type: error

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "rgsystem-%{[fields.app]}-%{+yyyy.MM.dd}"

setup.kibana:
  host: "kibana:5601"

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
```

### Application Logging

**Winston Configuration:**

```typescript
import winston from 'winston';
import { ElasticsearchTransport } from 'winston-elasticsearch';

const logFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  winston.format.errors({ stack: true }),
  winston.format.json()
);

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: logFormat,
  defaultMeta: {
    service: 'rgsystem-api',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION
  },
  transports: [
    // Console (for Docker logs)
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),

    // File - All logs
    new winston.transports.File({
      filename: '/var/log/rgsystem/api/combined.log',
      maxsize: 100 * 1024 * 1024, // 100MB
      maxFiles: 10,
      tailable: true
    }),

    // File - Errors only
    new winston.transports.File({
      filename: '/var/log/rgsystem/api/error.log',
      level: 'error',
      maxsize: 50 * 1024 * 1024, // 50MB
      maxFiles: 10
    }),

    // Elasticsearch
    new ElasticsearchTransport({
      level: 'info',
      clientOpts: {
        node: process.env.ELASTICSEARCH_URL || 'http://elasticsearch:9200'
      },
      index: 'rgsystem-api'
    })
  ],
  exceptionHandlers: [
    new winston.transports.File({
      filename: '/var/log/rgsystem/api/exceptions.log'
    })
  ],
  rejectionHandlers: [
    new winston.transports.File({
      filename: '/var/log/rgsystem/api/rejections.log'
    })
  ]
});

export default logger;
```

### Log Rotation

```bash
# /etc/logrotate.d/rgsystem
/var/log/rgsystem/**/*.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 rgsystem rgsystem
    sharedscripts
    postrotate
        systemctl reload rgsystem-api
    endscript
}
```

### Useful Log Queries

**Kibana Query Examples:**

```
# Find all errors in last 24 hours
level: "error" AND @timestamp:[now-24h TO now]

# Find slow queries (>1 second)
duration: >1000 AND query_type: "database"

# Find failed login attempts
event_type: "LOGIN_FAILURE" AND @timestamp:[now-1h TO now]

# Find 5xx errors by endpoint
status_code: [500 TO 599]

# Track specific user's actions
user_id: "abc-123-def" AND @timestamp:[now-1d TO now]

# WebSocket connection errors
component: "websocket" AND level: "error"

# High memory usage warnings
message: "high memory" AND level: "warn"
```

---

## Backup & Recovery

### Backup Strategy

**3-2-1 Backup Rule:**
- **3** copies of data
- **2** different storage media
- **1** off-site backup

### PostgreSQL Backup

**Automated Backup Script:**

```bash
#!/bin/bash
# /opt/scripts/backup-postgres.sh

set -e

# Configuration
BACKUP_DIR="/backup/postgres"
RETENTION_DAYS=30
S3_BUCKET="s3://rgsystem-backups"
DB_NAME="rgsystem_db"
DB_USER="postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/rgsystem_${TIMESTAMP}.sql.gz"

# Create backup directory
mkdir -p "${BACKUP_DIR}"

# Dump database
echo "[$(date)] Starting backup..."
pg_dump -U ${DB_USER} -d ${DB_NAME} | gzip > "${BACKUP_FILE}"

# Verify backup
if [ -f "${BACKUP_FILE}" ]; then
    SIZE=$(du -h "${BACKUP_FILE}" | cut -f1)
    echo "[$(date)] Backup created: ${BACKUP_FILE} (${SIZE})"
else
    echo "[$(date)] ERROR: Backup failed!"
    exit 1
fi

# Upload to S3
echo "[$(date)] Uploading to S3..."
aws s3 cp "${BACKUP_FILE}" "${S3_BUCKET}/postgres/" --storage-class STANDARD_IA

# Verify S3 upload
if aws s3 ls "${S3_BUCKET}/postgres/$(basename ${BACKUP_FILE})" > /dev/null; then
    echo "[$(date)] Successfully uploaded to S3"
else
    echo "[$(date)] ERROR: S3 upload failed!"
    exit 1
fi

# Remove local backups older than retention period
echo "[$(date)] Cleaning up old local backups..."
find "${BACKUP_DIR}" -name "rgsystem_*.sql.gz" -mtime +${RETENTION_DAYS} -delete

# Remove S3 backups older than 90 days
aws s3 ls "${S3_BUCKET}/postgres/" | while read -r line; do
    BACKUP_DATE=$(echo $line | awk '{print $1}')
    BACKUP_NAME=$(echo $line | awk '{print $4}')
    BACKUP_AGE=$(( ($(date +%s) - $(date -d "$BACKUP_DATE" +%s) )/(60*60*24) ))

    if [ $BACKUP_AGE -gt 90 ]; then
        echo "[$(date)] Deleting old S3 backup: ${BACKUP_NAME}"
        aws s3 rm "${S3_BUCKET}/postgres/${BACKUP_NAME}"
    fi
done

echo "[$(date)] Backup completed successfully"

# Send metrics to Prometheus Pushgateway
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/postgres_backup
# TYPE postgres_backup_success gauge
postgres_backup_success 1
# TYPE postgres_backup_size_bytes gauge
postgres_backup_size_bytes $(stat -c%s "${BACKUP_FILE}")
# TYPE postgres_backup_duration_seconds gauge
postgres_backup_duration_seconds ${SECONDS}
EOF
```

**Cron Schedule:**

```cron
# Full backup daily at 2 AM
0 2 * * * /opt/scripts/backup-postgres.sh >> /var/log/backups/postgres.log 2>&1

# Incremental WAL archiving (continuous)
* * * * * /opt/scripts/archive-wal.sh >> /var/log/backups/wal-archive.log 2>&1
```

### PostgreSQL Point-in-Time Recovery (PITR)

**WAL Archiving Configuration:**

```sql
-- postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /backup/wal_archive/%f && cp %p /backup/wal_archive/%f'
archive_timeout = 300  -- 5 minutes

-- Also sync to S3
-- archive_command = 'aws s3 cp %p s3://rgsystem-backups/wal/%f'
```

**Recovery Script:**

```bash
#!/bin/bash
# /opt/scripts/restore-postgres.sh

set -e

BACKUP_FILE=$1
TARGET_TIME=$2  # Optional: YYYY-MM-DD HH:MM:SS

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file> [target_time]"
    exit 1
fi

# Stop PostgreSQL
systemctl stop postgresql

# Backup current data directory
mv /var/lib/postgresql/14/main /var/lib/postgresql/14/main.old

# Restore base backup
gunzip -c "$BACKUP_FILE" | sudo -u postgres psql

# Configure recovery
if [ -n "$TARGET_TIME" ]; then
    cat > /var/lib/postgresql/14/main/recovery.conf <<EOF
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_time = '$TARGET_TIME'
recovery_target_action = 'promote'
EOF
fi

# Start PostgreSQL
systemctl start postgresql

echo "Recovery completed!"
```

### Redis Backup

```bash
#!/bin/bash
# /opt/scripts/backup-redis.sh

BACKUP_DIR="/backup/redis"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Trigger BGSAVE
redis-cli BGSAVE

# Wait for save to complete
while [ $(redis-cli LASTSAVE) -eq $(redis-cli LASTSAVE) ]; do
    sleep 1
done

# Copy RDB file
cp /var/lib/redis/dump.rdb "${BACKUP_DIR}/dump_${TIMESTAMP}.rdb"

# Upload to S3
aws s3 cp "${BACKUP_DIR}/dump_${TIMESTAMP}.rdb" s3://rgsystem-backups/redis/

echo "Redis backup completed"
```

### Application Files Backup

```bash
#!/bin/bash
# /opt/scripts/backup-files.sh

BACKUP_DIR="/backup/files"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/files_${TIMESTAMP}.tar.gz"

# Backup uploaded files
tar -czf "$BACKUP_FILE" /var/www/rgsystem/uploads

# Upload to S3
aws s3 cp "$BACKUP_FILE" s3://rgsystem-backups/files/

echo "Files backup completed"
```

### Backup Verification

```bash
#!/bin/bash
# /opt/scripts/verify-backup.sh

set -e

BACKUP_FILE=$1

# Test PostgreSQL backup restore
echo "Testing PostgreSQL backup restore..."
gunzip -c "$BACKUP_FILE" | sudo -u postgres psql -d rgsystem_test > /dev/null

# Run smoke tests
echo "Running smoke tests..."
psql -d rgsystem_test -c "SELECT COUNT(*) FROM users;" > /dev/null
psql -d rgsystem_test -c "SELECT COUNT(*) FROM competitions;" > /dev/null

# Cleanup
dropdb rgsystem_test

echo "Backup verification passed!"
```

---

## Performance Tuning

### PostgreSQL Tuning

```sql
-- /etc/postgresql/14/main/postgresql.conf

# Memory Settings
shared_buffers = 4GB              -- 25% of RAM
effective_cache_size = 12GB       -- 75% of RAM
work_mem = 64MB                   -- Per query operation
maintenance_work_mem = 1GB        -- For VACUUM, CREATE INDEX
wal_buffers = 16MB

# Checkpoint Settings
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB

# Query Planner
random_page_cost = 1.1            -- For SSD
effective_io_concurrency = 200    -- For SSD

# Connection Settings
max_connections = 200
shared_preload_libraries = 'pg_stat_statements'

# Logging
log_min_duration_statement = 1000  -- Log slow queries (>1s)
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
```

**Monitoring Query Performance:**

```sql
-- Enable pg_stat_statements
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slow queries
SELECT
    calls,
    total_time / 1000 AS total_seconds,
    mean_time / 1000 AS mean_seconds,
    query
FROM pg_stat_statements
WHERE mean_time > 1000  -- >1 second
ORDER BY mean_time DESC
LIMIT 20;

-- Find queries causing high I/O
SELECT
    query,
    calls,
    total_time,
    rows,
    100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS cache_hit_ratio
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;

-- Index usage statistics
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Table bloat check
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS external_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Redis Tuning

```conf
# /etc/redis/redis.conf

# Memory
maxmemory 3gb
maxmemory-policy allkeys-lru

# Persistence
save 900 1          # Save after 900 sec if 1 key changed
save 300 10         # Save after 300 sec if 10 keys changed
save 60 10000       # Save after 60 sec if 10000 keys changed

# Performance
tcp-backlog 511
timeout 300
tcp-keepalive 60

# Slow log
slowlog-log-slower-than 10000  # 10ms
slowlog-max-len 128
```

### Node.js Tuning

```bash
# Environment variables
NODE_ENV=production
NODE_OPTIONS="--max-old-space-size=4096"  # 4GB heap

# PM2 cluster mode
pm2 start ecosystem.config.js
```

**PM2 Configuration:**

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'rgsystem-api',
    script: './dist/server.js',
    instances: 'max',  // Use all CPU cores
    exec_mode: 'cluster',
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    error_file: '/var/log/rgsystem/api/pm2-error.log',
    out_file: '/var/log/rgsystem/api/pm2-out.log',
    merge_logs: true,
    autorestart: true,
    watch: false,
    max_restarts: 10,
    min_uptime: '10s'
  }]
};
```

### Nginx Tuning

```nginx
# /etc/nginx/nginx.conf

user www-data;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    # Basic
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    # Buffer sizes
    client_body_buffer_size 128k;
    client_max_body_size 10m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 16k;

    # Timeouts
    client_body_timeout 12;
    client_header_timeout 12;
    send_timeout 10;

    # Compression
    gzip on;
    gzip_comp_level 6;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_vary on;
    gzip_proxied any;

    # Caching
    open_file_cache max=200000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    upstream api_backend {
        least_conn;
        server 10.0.10.10:3000 max_fails=3 fail_timeout=30s;
        server 10.0.10.11:3000 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    server {
        listen 443 ssl http2;
        server_name rgsystem.local;

        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            limit_conn addr 10;

            proxy_pass http://api_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;

            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
            proxy_busy_buffers_size 8k;
        }
    }
}
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue: High Database CPU Usage

**Symptoms:**
- Database CPU consistently >80%
- Slow query responses
- Connection pool exhaustion

**Diagnosis:**

```sql
-- Find queries consuming most CPU time
SELECT
    pid,
    usename,
    application_name,
    state,
    query_start,
    NOW() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;

-- Check for locks
SELECT
    pg_stat_activity.pid,
    pg_stat_activity.usename,
    pg_locks.mode,
    pg_locks.granted,
    pg_stat_activity.query
FROM pg_stat_activity
JOIN pg_locks ON pg_locks.pid = pg_stat_activity.pid
WHERE NOT pg_locks.granted;
```

**Solutions:**

1. **Terminate long-running queries:**
```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
AND state = 'active'
AND NOW() - query_start > interval '5 minutes';
```

2. **Add missing indexes:**
```sql
-- Find unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0;

-- Find missing indexes
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n_distinct DESC;
```

3. **Optimize queries:**
```sql
EXPLAIN ANALYZE
SELECT * FROM athletes WHERE region = 'Moscow';
```

#### Issue: Memory Leak in Node.js

**Symptoms:**
- Increasing memory usage over time
- Process crashes with OOM
- GC taking longer

**Diagnosis:**

```bash
# Check heap usage
node --expose-gc --inspect server.js

# Connect Chrome DevTools
# chrome://inspect

# Generate heap snapshot
kill -USR2 <pid>
```

**Using heapdump:**

```javascript
import heapdump from 'heapdump';

// Automatically dump on OOM
heapdump.writeSnapshot((err, filename) => {
  console.log('Heap dump written to', filename);
});
```

**Solutions:**

1. **Increase heap size:**
```bash
NODE_OPTIONS="--max-old-space-size=4096"
```

2. **Find memory leaks:**
```javascript
// Use weak references for caches
const cache = new WeakMap();

// Clear event listeners
emitter.removeAllListeners('event');

// Close database connections
await pool.end();
```

3. **Restart periodically:**
```javascript
// PM2 max_memory_restart
max_memory_restart: '1G'
```

#### Issue: WebSocket Disconnections

**Symptoms:**
- Clients frequently disconnect
- Connection timeout errors
- High reconnection rate

**Diagnosis:**

```bash
# Check WebSocket connections
netstat -an | grep :3001 | wc -l

# Monitor connection duration
tail -f /var/log/rgsystem/api/websocket.log
```

**Solutions:**

1. **Implement heartbeat:**
```typescript
const WS_HEARTBEAT_INTERVAL = 30000; // 30 seconds

wss.on('connection', (ws) => {
  ws.isAlive = true;

  ws.on('pong', () => {
    ws.isAlive = true;
  });

  const interval = setInterval(() => {
    wss.clients.forEach((ws) => {
      if (ws.isAlive === false) {
        return ws.terminate();
      }

      ws.isAlive = false;
      ws.ping();
    });
  }, WS_HEARTBEAT_INTERVAL);

  ws.on('close', () => {
    clearInterval(interval);
  });
});
```

2. **Increase timeouts:**
```nginx
# Nginx
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```

3. **Load balancer sticky sessions:**
```nginx
upstream websocket_backend {
    ip_hash;  # Sticky sessions
    server 10.0.10.10:3001;
    server 10.0.10.11:3001;
}
```

---

## Runbooks

### Runbook: Database Failover

**When to execute:** Primary database failure

**Steps:**

1. **Verify primary is down:**
```bash
pg_isready -h db-primary -p 5432
# Expected: no response or connection refused
```

2. **Promote standby to primary:**
```bash
# On standby server
pg_ctl promote -D /var/lib/postgresql/14/main
```

3. **Update application config:**
```bash
# Update DATABASE_HOST in .env
DATABASE_HOST=db-standby.rgsystem.local

# Restart API servers
systemctl restart rgsystem-api
```

4. **Verify connectivity:**
```bash
psql -h db-standby -U rgsystem_user -d rgsystem_db -c "SELECT 1;"
```

5. **Monitor replication lag:**
```sql
SELECT NOW() - pg_last_xact_replay_timestamp() AS replication_lag;
```

6. **Post-incident:**
- Update DNS/load balancer
- Rebuild old primary as new standby
- Document incident

### Runbook: Clear Redis Cache

**When to execute:** Stale cache issues, after deployment

**Steps:**

1. **Check cache size:**
```bash
redis-cli INFO memory | grep used_memory_human
```

2. **Flush specific keys:**
```bash
# Delete keys matching pattern
redis-cli --scan --pattern "sessions:*" | xargs redis-cli DEL

# Flush entire cache (CAUTION!)
redis-cli FLUSHALL
```

3. **Verify application:**
```bash
curl -I https://rgsystem.local/api/health
# Should return 200 OK
```

4. **Monitor cache hit rate:**
```bash
redis-cli INFO stats | grep keyspace
```

### Runbook: Scale API Servers

**When to execute:** High load, approaching capacity

**Steps:**

1. **Check current load:**
```bash
# API request rate
curl -s http://prometheus:9090/api/v1/query?query=rate(http_requests_total[5m])

# CPU usage
top -b -n 1 | grep node
```

2. **Deploy new API instance:**
```bash
# On new server
git clone https://github.com/rgsystem/api.git
cd api
npm install --production
pm2 start ecosystem.config.js
```

3. **Add to load balancer:**
```nginx
# Edit /etc/nginx/conf.d/upstream.conf
upstream api_backend {
    server 10.0.10.10:3000;
    server 10.0.10.11:3000;
    server 10.0.10.12:3000;  # New server
}
```

4. **Reload Nginx:**
```bash
nginx -t && nginx -s reload
```

5. **Monitor new instance:**
```bash
# Check health
curl http://10.0.10.12:3000/health

# Monitor metrics
watch -n 1 'curl -s http://10.0.10.12:9090/metrics | grep http_requests_total'
```

---

## Maintenance Windows

### Scheduled Maintenance

**Schedule:** Every 2nd Sunday of the month, 02:00-04:00 UTC

**Maintenance Checklist:**

```markdown
## Pre-Maintenance (T-24h)
- [ ] Announce maintenance window to users
- [ ] Create backup of current state
- [ ] Test rollback procedures
- [ ] Prepare runbooks
- [ ] Brief on-call team

## During Maintenance
- [ ] Enable maintenance mode
- [ ] Stop accepting new requests
- [ ] Drain existing connections
- [ ] Perform updates
- [ ] Run database migrations
- [ ] Clear caches
- [ ] Restart services
- [ ] Run smoke tests
- [ ] Disable maintenance mode

## Post-Maintenance
- [ ] Monitor error rates (15min)
- [ ] Check performance metrics (15min)
- [ ] Verify all features working
- [ ] Update documentation
- [ ] Send completion notification
```

---

## Disaster Recovery

### Recovery Time Objective (RTO): 1 hour
### Recovery Point Objective (RPO): 15 minutes

### Disaster Scenarios

#### Scenario 1: Complete Data Center Failure

**Recovery Steps:**

1. **Failover to backup region** (if multi-region)
2. **Restore from S3 backups:**
```bash
# Restore PostgreSQL
aws s3 cp s3://rgsystem-backups/postgres/latest.sql.gz /tmp/
gunzip -c /tmp/latest.sql.gz | psql -d rgsystem_db

# Restore Redis
aws s3 cp s3://rgsystem-backups/redis/latest.rdb /var/lib/redis/dump.rdb
systemctl restart redis
```

3. **Update DNS to point to DR site**
4. **Verify all services operational**

#### Scenario 2: Data Corruption

**Recovery Steps:**

1. **Identify corruption extent:**
```sql
SELECT * FROM corrupted_table LIMIT 10;
```

2. **Restore from PITR:**
```bash
/opt/scripts/restore-postgres.sh /backup/latest.sql.gz "2025-11-27 14:30:00"
```

3. **Verify data integrity:**
```sql
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM competitions;
```

---

## Capacity Planning

### Growth Projections

| Metric | Current | 6 Months | 12 Months |
|--------|---------|----------|-----------|
| Users | 1,000 | 5,000 | 10,000 |
| Competitions/month | 50 | 200 | 500 |
| Database Size | 50 GB | 200 GB | 500 GB |
| API RPS | 100 | 500 | 1,000 |

### Scaling Triggers

**Horizontal Scaling (Add Servers):**
- CPU usage >70% for 1 hour
- Request queue depth >100
- Response time p95 >300ms for 15 minutes

**Vertical Scaling (Bigger Servers):**
- Memory usage >85% consistently
- Database connection pool >80% utilized

**Database Scaling:**
- Replication lag >30 seconds
- Disk I/O wait >20%
- Active connections >150

---

## On-Call Procedures

### On-Call Rotation

- **Primary:** Week rotation
- **Secondary:** Week rotation
- **Escalation:** CTO

### Alert Response SLA

| Severity | Response Time | Resolution Time |
|----------|---------------|-----------------|
| Critical | 15 minutes | 1 hour |
| High | 1 hour | 4 hours |
| Medium | 4 hours | 1 business day |
| Low | 1 business day | 3 business days |

### Incident Management

1. **Acknowledge alert** in PagerDuty
2. **Assess severity** using runbook
3. **Communicate** in #incidents Slack channel
4. **Mitigate** using appropriate runbook
5. **Document** actions taken
6. **Resolve** and close incident
7. **Post-mortem** within 48 hours (for Critical/High)

---

**Last Updated:** 2025-11-27
**Review Schedule:** Quarterly
**Document Owner:** DevOps Team

---

> **Note:** This operations manual should be kept up-to-date with production changes. All runbooks should be tested regularly during maintenance windows.
