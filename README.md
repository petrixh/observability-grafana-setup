# Vaadin Observability Grafana Setup

This repo provides a docker compose setup for collecting traces, metrics and logs from a Vaadin application instrumented with the Vaadin Observability Agent into Grafana.  

> **Warning**
> This setup is only intended as a local test setup. It is not production ready.

To start the setup, just run:
```
docker compose up
```

The setup runs the OpenTelemetry Collector which exposes the following endpoints for receiving data:
- `http://localhost:4317` (OTLP GRPC)
- `http://localhost:4318` (OTLP HTTP)

To configure the agent to send data to this setup, create an `agent.properties` file with the following contents:
```
otel.service.name=vaadin
otel.traces.exporter=otlp
otel.metrics.exporter=otlp
otel.logs.exporter=otlp
```

Then start your Vaadin app together with the agent, for example:
```
java -javaagent:observability-kit-agent-4.0.0.jar \
     -Dotel.javaagent.configuration-file=agent.properties \
     -jar myapp.jar
```

The Grafana UI is available at http://localhost:3000. Anonymous access with admin privileges is enabled by default, so no login is required.
The setup also provides sample dashboards under the **Vaadin** folder, including a dashboard for Observability Kit 3.1.0.

To stop all services in this setup, run:
```
docker compose stop
```

To remove all services and volumes from this setup, run:
```
docker compose down -v
```

## Services

A quick overview of the services run by this setup, and how they interact with each other.

### OpenTelemetry Collector

The collector is configured to receive trace, metrics and log data in the `OTLP` format either through `GRPC` (port 4317) or `HTTP` (port 4318).

It then distributes that data to individual services that are then used by Grafana to query data from:
- Traces are sent to Grafana Tempo via OTLP HTTP
- Metrics are exposed to be scraped by Prometheus
- Logs are sent to Grafana Loki via its native OTLP endpoint

### Grafana Tempo

Receives traces from the OpenTelemetry Collector and stores them.

Exposes a query API that is used by Grafana to search for traces.

### Prometheus

Scrapes metrics from an endpoint provided by the OpenTelemetry Collector and stores them.

Also receives remote-write data from Tempo's metrics generator (service graphs and span metrics).

Exposes a query API that is used by Grafana to search for metrics.

### Loki

Receives logs from the OpenTelemetry Collector and stores them.

Exposes a query API that is used by Grafana to search for logs.

### Grafana

Provides the UI to display the traces, metrics and logs. The Grafana setup automatically provisions data sources for collecting the respective data from Tempo, Prometheus and Loki. It also includes a basic dashboard for showing some metrics, traces and logs - this requires the OpenTelemetry service name to be configured as `vaadin`, which is the default when using the Vaadin Observability agent.

## Data Retention

Each service manages its own data retention independently. Since this is a local test setup, the defaults are tuned for convenience rather than long-term storage. All data is stored in Docker volumes and can be wiped with `docker compose down -v`.

### Tempo (traces)

Tempo does not have an explicit retention period configured, so trace data is kept until disk space runs out. The WAL and blocks are stored in the `tempo_data` volume under `/var/tempo`.

To configure a retention period, add a `compaction` block to `tempo/tempo.yaml`:
```yaml
compactor:
  compaction:
    block_retention: 48h        # how long to keep completed blocks
```

### Prometheus (metrics)

Prometheus uses its default retention of **15 days** (`--storage.tsdb.retention.time=15d`). There is no size-based limit configured.

To change the retention period, add a flag to the `command` list in `docker-compose.yml`:
```yaml
prometheus:
  command:
    - "--storage.tsdb.retention.time=7d"        # keep metrics for 7 days
    - "--storage.tsdb.retention.size=1GB"        # or cap storage at 1GB
    - "--web.enable-remote-write-receiver"
    - "--config.file=/etc/prometheus/prometheus.yml"
```

### Loki (logs)

Loki uses its built-in default configuration with no explicit retention. Log data is kept indefinitely in the `loki` container's local filesystem (not in a named volume, so it is lost when the container is recreated).

To configure log retention, mount a custom Loki config and enable the compactor's retention feature:
```yaml
limits_config:
  retention_period: 72h         # minimum retention period

compactor:
  retention_enabled: true
  delete_request_store: filesystem
```

### Grafana

Grafana stores its own configuration database (dashboards, preferences) in the `grafana_data` volume. This is not telemetry data and does not require retention management. Provisioned dashboards are loaded from the mounted JSON files on each startup.
