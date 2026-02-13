# Build stage: Get busybox
FROM busybox:1.37.0 AS busybox_builder

# Main image
FROM grafana/loki:3.6.5 AS base

# Copy busybox from builder stage
COPY --from=busybox_builder /bin/busybox /bin/busybox

# Labels
LABEL description="MOV.AI Log Aggregator - Grafana Loki based log aggregator and forwarder" \
      maintainer="devops@mov.ai" \
      org.opencontainers.image.source="https://github.com/MOV-AI/containers-log-aggregator" \
      org.opencontainers.image.documentation="https://github.com/MOV-AI/containers-log-aggregator/blob/main/README.md" \
      org.opencontainers.image.vendor="MOV.AI" \
      org.opencontainers.image.title="MOV.AI Log Aggregator"

# Configuration Files: YAML Format
# - loki-config.yml: Configuration for Grafana Loki
COPY files/*.yml /etc/loki/loki-config.yml

# Environment Variables
ENV LOG_LEVEL=info
ENV DEVICE_NAME=manager
ENV APP_NAME=default
ENV LOKI_HOST=loki
ENV LOKI_PORT=3100

#HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 CMD ["/bin/busybox", "pidof", "loki"]
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 CMD ["/usr/bin/loki", "-version"]
