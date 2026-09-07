# Vaadin Observability Grafana Setup

This repo provides a docker compose setup for collecting traces and metrics from a Vaadin application instrumented with Vaadin Observability Kit into Grafana.  

> **Warning**
> This setup is only intended as a local test setup. It is not production ready.

To start the setup, just run:
```
docker compose up
```

The setup runs the OpenTelemetry Collector which exposes the following endpoints for receiving data:
- `http://localhost:4317` (OTLP GRPC)
- `http://localhost:4318` (OTLP HTTP)

### Sending data from Observability Kit 5

Observability Kit 5 is a Micrometer library rather than a Java agent: there is no `-javaagent`
flag and no `otel.*` properties. Add the kit plus Spring Boot's OpenTelemetry starter to your
app:

```xml
<dependency>
    <groupId>com.vaadin</groupId>
    <artifactId>observability-kit-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-opentelemetry</artifactId>
</dependency>
```

and point its OTLP exporters at this setup's collector:

```properties
# The dashboard filters on service_name="vaadin"
spring.application.name=vaadin
management.otlp.metrics.export.url=http://localhost:4318/v1/metrics
management.otlp.metrics.export.step=10s
management.opentelemetry.tracing.export.otlp.endpoint=http://localhost:4318/v1/traces
# Default is 0.1 - sample everything so a demo does not look broken
management.tracing.sampling.probability=1.0
```

Then just run your app normally:
```
java -jar myapp.jar
```

#### Push vs scrape

The properties above **push** metrics over OTLP to the collector, which re-exposes them for
Prometheus on port 8090 (see `prometheus/prometheus.yml`). The alternative is to let Prometheus
**scrape** the app directly: add `io.micrometer:micrometer-registry-prometheus` to the app,
expose the endpoint with `management.endpoints.web.exposure.include=prometheus`, and add a
second target to `prometheus/prometheus.yml`:

```yaml
  - job_name: 'vaadin-app'
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['host.docker.internal:8080']
```

That target is deliberately **not** enabled here - the dashboard is built against the pushed
metrics, whose labels carry `exported_job="vaadin"` from the collector.

Observability Kit 5 no longer exports logs, so nothing reaches Loki out of the box. Loki is
still part of this setup; wiring the OpenTelemetry Logback appender to it is left as an exercise.

The Grafana UI is available at http://localhost:3000. Anonymous access with admin privileges is enabled by default, so no login is required.
The setup provisions **Vaadin Dashboard - 5.0.0** (uid `vaadin-obskit-5`) under the **Vaadin**
folder. The older `vaadin-dashboard.json`, `vaadin-dashboard-3.1.0.json` and
`vaadin-dashboard-4.0.0.json` files are kept in the repo for reference but are no longer mounted:
Observability Kit 5 renamed every meter (`vaadin.session.count` -> `vaadin.sessions.active`,
`vaadin.ui.count` -> `vaadin.ui.active`, JVM metrics now come from Spring Boot Actuator as
`jvm_memory_used_bytes` / `process_cpu_usage`), so those dashboards can only render "No data".

Note that the 5.0.0 dashboard contains no `histogram_quantile()` panels. Micrometer timers
publish count/sum/max only; the OTLP registry emits a single `+Inf` bucket unless percentile
histograms are enabled per meter, so latency panels are built on
`rate(_sum[1m]) / rate(_count[1m])` and the `_max` gauge instead.

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
- Logs are sent to Grafana Loki via its native OTLP endpoint (Observability Kit 5 does not export logs, so this pipeline is idle unless the app sends its own)

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

Provides the UI to display the traces, metrics and logs. The Grafana setup automatically provisions data sources for collecting the respective data from Tempo, Prometheus and Loki. It also includes a basic dashboard for showing some metrics and traces - this requires the OpenTelemetry service name to be configured as `vaadin`, which is what `spring.application.name=vaadin` gives you with Observability Kit 5.

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
