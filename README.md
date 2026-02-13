# MOV.AI Log Aggregator

A centralized log aggregation service for the MOV.AI platform, built on [Grafana Loki](https://grafana.com/oss/loki/). This service receives logs from distributed Log Agents, indexes them, and provides a queryable interface for analysis and visualization.

## Overview

The Log Aggregator is a critical component of MOV.AI's observability stack. It receives logs from multiple Log Agent instances running on manager and worker nodes, stores them efficiently, and provides a robust API for querying and visualization through Grafana.

### Key Features

- **Distributed Log Collection**: Aggregates logs from multiple sources via Log Agent
- **Efficient Storage**: Uses label-based indexing for fast queries with minimal storage overhead
- **Multi-tenancy Support**: Isolate logs by job, source, and custom labels
- **High Availability**: Designed for scalability and redundancy
- **Query API**: RESTful API compatible with Grafana Loki protocol
- **Time Series Optimization**: Excellent performance for time-series log queries
- **Integration Ready**: Works seamlessly with Grafana, Prometheus, and Alertmanager

## Architecture

```
┌─────────────────────────────────────────────────┐
│         Log Sources (Manager & Workers)         │
│  ├─ Log Agent Manager                           │
│  ├─ Log Agent Worker 1                          │
│  ├─ Log Agent Worker 2                          │
│  └─ Log Agent Worker N                          │
└────────────┬────────────────────────────────────┘
             │ Forward Protocol (Port 24224)
             │ (via Log Agents)
┌────────────▼────────────────────────────────────┐
│   Log Aggregator (Loki)                         │
│  ├─ Distributor (Log Ingestion)                 │
│  ├─ Ingester (Buffering & WAL)                  │
│  ├─ Querier (Log Retrieval)                     │
│  ├─ Index Store (Label Index)                   │
│  └─ Chunk Store (Log Data)                      │
└────────────┬────────────────────────────────────┘
             │
    ┌────────┴──────────┐
    │                   │
┌───▼─────┐      ┌──────▼──────┐
│ Grafana │      │ Query API   │
│ UI      │      │ (REST/gRPC) │
└─────────┘      └─────────────┘
```

## Image Information

- **Base Image**: TBD (Grafana Loki distribution)
- **Platforms**: `linux/amd64, linux/arm/v7, linux/arm64`
- **Registry Image**: `qa/log-aggregator`
- **Public Image**: `ce/log-aggregator`
- **Status**: In Development

## Prerequisites

### Minimum Requirements

- **CPU**: 2 cores minimum (4+ recommended for production)
- **Memory**: 2GB minimum (4GB+ recommended)
- **Storage**: Persistent volume for log chunks and index
- **Networking**: Network connectivity to all Log Agent instances

### Storage Requirements

Estimated storage usage:
- **Per Day** (assuming 1GB logs): 100-200MB (with compression and indexing)
- **Per Month**: 3-6GB
- **Recommended**: Allocate 10x+ log volume for index and growth headroom

## Configuration

### Docker Compose Example

```yaml
version: '3.8'

services:
  loki:
    image: ce/log-aggregator:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./config/loki-config.yml:/etc/loki/cli-config.yml:ro
      - loki-storage:/loki
    environment:
      - LOKI_LOG_LEVEL=info
    networks:
      - logging
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3100/loki/api/v1/status/buildinfo"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  # Optional: Grafana Dashboard
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-storage:/var/lib/grafana
      - ./config/loki-datasource.json:/etc/grafana/provisioning/datasources/loki.json:ro
    depends_on:
      - loki
    networks:
      - logging

volumes:
  loki-storage:
  grafana-storage:

networks:
  logging:
    driver: bridge
```

### Configuration File (`loki-config.yml`)

```yaml
auth_enabled: false

ingester:
  chunk_idle_period: 3m
  max_chunk_age: 1h
  chunk_retain_period: 1m
  max_streams_per_user: 10000
  lifecycler:
    ring:
      kvstore:
        store: inmemory

# Optional: Index Cache
index_cache_validity: 5m

# Storage Configuration
schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

# Query Configuration
chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 30d

# Server Configuration
server:
  http_listen_port: 3100
  log_level: info

# Querier Configuration
querier:
  max_concurrent: 20
  query_timeout: 1m
```

## Usage

### Build

To build the Docker image locally:

```bash
docker build -t registry.cloud.mov.ai/qa/log-aggregator:latest -f docker/Dockerfile .
```
### Docker Run Example

```bash
docker run -d \
  --name loki \
  -v $(pwd)/loki-config.yml:/etc/loki/cli-config.yml:ro \
  -v loki-storage:/loki \
  -p 3100:3100 \
  ce/log-aggregator:latest
```

### Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loki
  namespace: logging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: loki
  template:
    metadata:
      labels:
        app: loki
    spec:
      containers:
      - name: loki
        image: ce/log-aggregator:1.0.0
        ports:
        - containerPort: 3100
          name: http-metrics
        volumeMounts:
        - name: config
          mountPath: /etc/loki
        - name: storage
          mountPath: /loki
        livenessProbe:
          httpGet:
            path: /loki/api/v1/status/buildinfo
            port: http-metrics
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /loki/api/v1/status/buildinfo
            port: http-metrics
          initialDelaySeconds: 10
          periodSeconds: 10
      volumes:
      - name: config
        configMap:
          name: loki-config
      - name: storage
        persistentVolumeClaim:
          claimName: loki-storage
---
apiVersion: v1
kind: Service
metadata:
  name: loki
  namespace: logging
spec:
  selector:
    app: loki
  ports:
  - port: 3100
    name: http
  type: ClusterIP
```

## API Reference

### Ingestion API

**Endpoint**: `POST /loki/api/v1/push`

Receive log entries from Log Agent instances.

```bash
curl -X POST http://localhost:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -d '{
    "streams": [
      {
        "stream": {
          "job": "manager",
          "source": "docker",
          "container": "spawner"
        },
        "values": [
          ["1234567890000000000", "log message here"]
        ]
      }
    ]
  }'
```

### Query API

**Endpoint**: `GET /loki/api/v1/query`

Query logs using LogQL syntax.

```bash
# Query all logs from manager job
curl 'http://localhost:3100/loki/api/v1/query?query={job="manager"}'

# Query with time range
curl 'http://localhost:3100/loki/api/v1/query_range?query={job="worker"}&start=1234567890&end=1234567900&step=60s'

# Query with regex filter
curl 'http://localhost:3100/loki/api/v1/query?query={job="manager"}%20%7C%20ERROR'
```

### Status API

**Endpoint**: `GET /loki/api/v1/status/buildinfo`

Check service health and build information.

```bash
curl http://localhost:3100/loki/api/v1/status/buildinfo
```

## Monitoring

### Health Checks

```bash
# Readiness check
curl http://localhost:3100/loki/api/v1/status/buildinfo

# Metrics endpoint (if enabled)
curl http://localhost:3100/metrics
```

### Key Metrics to Monitor

- `loki_distributor_received_lines_total`: Total lines received
- `loki_ingester_chunks_created_total`: Chunks created
- `loki_querier_request_duration_seconds`: Query latency
- `loki_ingester_memory_chunks`: In-memory chunks
- `loki_ingester_wal_truncate_duration_seconds`: WAL performance

### Example Prometheus Queries

```promql
# Ingestion rate (lines/sec)
rate(loki_distributor_received_lines_total[5m])

# Logs by job
sum by (job) (rate(loki_distributor_received_lines_total[5m]))

# Query performance
histogram_quantile(0.95, rate(loki_querier_request_duration_seconds_bucket[5m]))
```

## Data Retention

### Automatic Cleanup

Configure retention in `loki-config.yml`:

```yaml
table_manager:
  retention_deletes_enabled: true
  retention_period: 30d  # Keep logs for 30 days
```

### Manual Cleanup

```bash
# Delete logs older than 7 days (requires admin access)
# Use Loki's retention API or external tools like logcli
```

## Performance Tuning

### For High Volume (>1000 logs/sec)

```yaml
ingester:
  chunk_idle_period: 5m          # Longer batching
  max_chunk_age: 2h              # Larger chunks
  max_streams_per_user: 50000    # Higher stream limit

querier:
  max_concurrent: 50             # More concurrent queries
```

### For Limited Resources (<2GB RAM)

```yaml
ingester:
  chunk_idle_period: 1m          # More frequent flushing
  max_chunk_age: 30m             # Smaller chunks
  max_streams_per_user: 1000     # Limited streams

querier:
  max_concurrent: 5              # Fewer concurrent queries
```

## Troubleshooting

### High Memory Usage

**Issue**: Loki pod/container consuming excessive memory

**Debug**:
```bash
# Check metrics
curl http://localhost:3100/metrics | grep loki_ingester_memory

# Check stream count
curl 'http://localhost:3100/loki/api/v1/labels/job/values'
```

**Solutions**:
- Increase `max_chunk_age` to batch logs longer
- Decrease `max_streams_per_user` if too many unique label combinations
- Add more memory to the container

### Slow Queries

**Issue**: Query responses taking >10 seconds

**Solutions**:
```yaml
# Enable index caching
index_cache_validity: 5m

# Optimize retention period
retention_period: 7d  # Smaller window

# Increase querier resources
querier:
  max_concurrent: 20
```

### Missing Logs

**Issue**: Logs not appearing in Loki

**Verification Steps**:
```bash
# 1. Check Loki is receiving data
curl 'http://localhost:3100/loki/api/v1/query?query={job="manager"}'

# 2. Verify Log Agent is connected and sending
docker logs log-agent-manager | grep -E "ERROR|sent"

# 3. Check network connectivity
docker exec loki curl -v http://log-agent-manager:24224

# 4. Review ingester status
curl http://localhost:3100/loki/api/v1/status/buildinfo
```

## File Structure

```
containers-log-aggregator/
├── docker/
│   └── Dockerfile              # Loki base image and configuration
├── .github/
│   ├── dependabot.yml          # Dependency updates
│   └── workflows/
│       ├── docker-ci.yml       # CI/CD pipeline
│       └── autoupdate.yml      # Auto-update dependencies
├── .dockerignore                # Docker build exclusions
├── .pre-commit-config.yaml      # Pre-commit hooks
├── .bumpversion.toml            # Version management
├── images-manifest.yml          # Container registry manifest
└── README.md                    # This file
```

## Building

### Build Locally

```bash
# Build with default target (when Dockerfile is created)
docker build -t log-aggregator:local docker/

# Build with specific platform
docker buildx build \
  --platform linux/amd64 \
  -t log-aggregator:local \
  docker/
```

## Security Considerations

1. **Network Isolation**: Restrict inbound traffic to Log Agents only
2. **TLS Support**: Use reverse proxy (nginx/Traefik) for HTTPS
3. **Authentication**: Configure OAuth/LDAP for Grafana access
4. **Data Protection**: Encrypt storage volumes at rest
5. **Access Control**: Implement RBAC for multi-tenant scenarios

```bash
# Example: Restrict to local network only
docker run -d \
  --name loki \
  -v $(pwd)/loki-config.yml:/etc/loki/cli-config.yml:ro \
  -v loki-storage:/loki \
  -p 127.0.0.1:3100:3100 \
  ce/log-aggregator:1.0.0
```

## Integration with Log Agent

The Log Aggregator is designed to work seamlessly with the **Log Agent**:

1. **Log Agent** (Fluent Bit) collects logs from manager and worker nodes
2. **Log Agent** forwards logs to **Log Aggregator** via Loki push API
3. **Log Aggregator** indexes and stores logs
4. **Grafana** queries **Log Aggregator** for visualization

Configuration example in Log Agent:

```properties
[OUTPUT]
    Name   loki
    Match  docker.*
    Host   log-aggregator  # Must be resolvable from agent
    Port   3100
    Labels job=manager,source=docker
```

## Related Services

- **Log Agent**: Collects and forwards logs to this service
- **Grafana**: Provides UI for querying and visualizing logs
- **Prometheus**: Optional metrics collection from Loki

## License

[MOV.AI License](../LICENSE)

## Additional Resources

- [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)
- [LogQL Query Language](https://grafana.com/docs/loki/latest/logql/)
- [Loki Configuration Reference](https://grafana.com/docs/loki/latest/configuration/)
- [MOV.AI Service Repository](https://github.com/MOV-AI/movai-service)
