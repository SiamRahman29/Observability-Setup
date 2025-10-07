# Observability Setup

A complete observability stack using Grafana LGTM (Loki, Grafana, Tempo, Mimir) with OpenTelemetry Collector and MinIO as local object storage.

## Stack Components

- **Grafana**: Visualization and dashboards (port 3000)
- **Loki**: Log aggregation system
- **Tempo**: Distributed tracing backend
- **Mimir**: Prometheus-compatible metrics storage
- **Prometheus**: Metrics collection and federation
- **OpenTelemetry Collector**: Telemetry data collection
- **MinIO**: S3-compatible object storage (ports 9000/9001)
- **cAdvisor**: Container metrics
- **Memcached**: Query acceleration cache

**Note:** This setup runs its own MinIO instance on ports 9002/9003 (separate from your app's MinIO).

## Prerequisites

- Docker and Docker Compose installed
- At least 4GB of available RAM
- Ports 3000, 3100, 3200, 4317, 4318, 8080, 9002, 9003, 9009, 9092 available

## Quick Start

### 1. Environment Setup

Create your environment file:

```bash
cp .env.example .env
```

The default `.env` file is pre-configured for local MinIO. You can keep it as-is or modify the bucket names if needed.

### 2. Start All Services

```bash
docker-compose up -d
```

Wait for all services to start (typically 30-60 seconds).

### 3. Create MinIO Buckets

Access MinIO Console at **http://localhost:9003**

- Username: `minioadmin`
- Password: `minioadmin`

Create the following buckets:
1. `observability` - for Loki logs and Tempo traces
2. `mimir-ruler` - for Mimir metrics and ruler storage

**Via MinIO Console:**
- Click "Buckets" in the left sidebar
- Click "Create Bucket" button
- Enter bucket name: `observability`
- Click "Create Bucket"
- Repeat for `mimir-ruler` bucket

**Via Docker CLI (Alternative):**

```bash
# Create buckets using MinIO client
docker exec -it observability-minio mc mb /data/observability
docker exec -it observability-minio mc mb /data/mimir-ruler

# Verify buckets were created
docker exec -it observability-minio mc ls /data/
```

### 4. Restart Services

After creating the buckets, restart the services that depend on them:

```bash
docker-compose restart loki mimir tempo
```

### 5. Verify Services

Check that all services are running:

```bash
docker-compose ps
```

All services should show "Up" status.

Check service logs to ensure they connected to MinIO successfully:

```bash
docker-compose logs loki | grep -i "s3\|error"
docker-compose logs mimir | grep -i "s3\|error"
docker-compose logs tempo | grep -i "s3\|error"
```

## Access URLs

| Service | URL | Purpose |
|---------|-----|---------|
| **Grafana** | http://localhost:3000 | Dashboards and visualization |
| **MinIO Console** | http://localhost:9003 | Object storage management |
| **MinIO API** | http://localhost:9002 | MinIO S3-compatible API |
| **Prometheus** | http://localhost:9092 | Metrics exploration |
| **Loki** | http://localhost:3100 | Log queries |
| **Tempo** | http://localhost:3200 | Trace queries |
| **Mimir** | http://localhost:9009 | Metrics ingestion |
| **cAdvisor** | http://localhost:8080 | Container metrics |

**Note:** 
- Grafana is configured with anonymous access (Admin role) for easy local development.
- This MinIO (9002/9003) is separate from your application's MinIO.

## Application Integration

### OpenTelemetry Endpoints

Configure your application to send telemetry data to these endpoints:

#### **Traces and Logs (OTLP Protocol)**

**gRPC (Recommended):**
```
Endpoint: localhost:4317
Protocol: gRPC
```

**HTTP:**
```
Endpoint: http://localhost:4318
Protocol: HTTP
```

### Configuration Examples

#### Node.js (OpenTelemetry SDK)

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { OTLPLogExporter } = require('@opentelemetry/exporter-logs-otlp-grpc');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'grpc://localhost:4317',
  }),
  logExporter: new OTLPLogExporter({
    url: 'grpc://localhost:4317',
  }),
  serviceName: 'your-service-name',
});

sdk.start();
```

#### Python (OpenTelemetry)

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Setup tracing
trace.set_tracer_provider(TracerProvider())
otlp_exporter = OTLPSpanExporter(endpoint="localhost:4317", insecure=True)
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(otlp_exporter))
```

#### Go (OpenTelemetry)

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/trace"
)

exporter, err := otlptracegrpc.New(
    context.Background(),
    otlptracegrpc.WithEndpoint("localhost:4317"),
    otlptracegrpc.WithInsecure(),
)

tp := trace.NewTracerProvider(
    trace.WithBatcher(exporter),
)
otel.SetTracerProvider(tp)
```

#### Java (OpenTelemetry)

```java
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;

OtlpGrpcSpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://localhost:4317")
    .build();

SdkTracerProvider sdkTracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(spanExporter).build())
    .build();

OpenTelemetrySdk.builder()
    .setTracerProvider(sdkTracerProvider)
    .build();
```

### Metrics Collection

For metrics, you have two options:

#### Option 1: Prometheus Client Libraries (Push to Mimir)

Configure your Prometheus client to remote write to:
```
URL: http://localhost:9009/api/v1/push
```

#### Option 2: Prometheus Scraping

Add your application as a scrape target in `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: "your-app"
    scrape_interval: 10s
    static_configs:
      - targets: ["host.docker.internal:YOUR_METRICS_PORT"]
```

Then restart Prometheus:
```bash
docker-compose restart prometheus
```

### Environment Variables Summary

Set these in your application:

```bash
# For OTLP (Traces & Logs)
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Or for HTTP
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Service identification
OTEL_SERVICE_NAME=your-service-name
OTEL_RESOURCE_ATTRIBUTES=environment=development,version=1.0.0
```

## Viewing Your Data in Grafana

1. **Access Grafana** at http://localhost:3000

2. **Data Sources** (pre-configured in provisioning):
   - Loki (Logs)
   - Tempo (Traces)
   - Mimir (Metrics)
   - Prometheus (Metrics)

3. **Explore Your Data:**
   - Click "Explore" in the left sidebar
   - Select data source (Loki, Tempo, or Mimir)
   - Query your telemetry data

4. **Create Dashboards:**
   - Click "+" → "Dashboard"
   - Add panels with queries
   - Save your dashboard

## Common Commands

### Start Services
```bash
docker-compose up -d
```

### Stop Services
```bash
docker-compose down
```

### View Logs
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f loki
docker-compose logs -f tempo
docker-compose logs -f mimir
```

### Restart a Service
```bash
docker-compose restart <service-name>
```

### Rebuild and Restart
```bash
docker-compose up -d --build
```

### Check Service Health
```bash
docker-compose ps
```

### Clean Up Everything (including volumes)
```bash
docker-compose down -v
```

## Troubleshooting

### Services Won't Start

Check logs:
```bash
docker-compose logs <service-name>
```

### Buckets Not Found Errors

Make sure you created the MinIO buckets:
```bash
# Check buckets exist
docker exec -it observability-minio mc ls /data/

# You should see:
# [date] observability/
# [date] mimir-ruler/

# If buckets are missing, create them:
docker exec -it observability-minio mc mb /data/observability
docker exec -it observability-minio mc mb /data/mimir-ruler

# Restart observability services
docker-compose restart loki mimir tempo
```

### Can't See Data in Grafana

1. Verify your application is sending data:
   ```bash
   docker-compose logs otel-collector
   ```

2. Check service endpoints are reachable:
   ```bash
   curl http://localhost:4318/v1/traces
   curl http://localhost:3100/ready
   curl http://localhost:3200/ready
   ```

3. Verify data is being ingested:
   ```bash
   # Check Loki
   curl http://localhost:3100/metrics | grep loki_ingester_streams

   # Check Tempo
   curl http://localhost:3200/metrics | grep tempo_ingester_traces
   ```

### Port Conflicts

If ports are already in use, you can modify them in `docker-compose.yml`:
```yaml
ports:
  - "NEW_PORT:CONTAINER_PORT"
```

### Memory Issues

If containers are OOMKilled, increase Docker's memory limit:
- Docker Desktop: Settings → Resources → Memory
- Linux: Modify Docker daemon configuration

## Data Persistence

The following data is persisted in Docker volumes:

- `mimir-data`: Mimir local metrics cache
- `tempo-data`: Tempo local traces cache
- `minio-data`: MinIO object storage (logs, metrics, traces)

To completely reset and delete all data:
```bash
docker-compose down -v
```

**Warning:** This will delete all observability data including historical logs, metrics, and traces stored in MinIO.

## Architecture

```
Your Application
    ↓
    ↓ (OTLP: gRPC/HTTP)
    ↓
OpenTelemetry Collector
    ↓
    ├── Logs    → Loki    ──┐
    ├── Traces  → Tempo   ──┤
    └── Metrics → Mimir   ──┤
                     ↑      │
                     │      │
                Prometheus  │
                     ↑      │
                     │      │
                cAdvisor    │
                            ↓
                    MinIO (S3 Storage)
                      (ports 9002/9003)
    
All visible via Grafana (port 3000)

Note: Your app's MinIO is separate from this observability MinIO (9002/9003)
```

## Production Considerations

This setup is optimized for local development. For production:

1. **Enable Authentication**: Configure Grafana authentication
2. **Use External Object Storage**: Replace MinIO with AWS S3, GCS, or Azure Blob Storage
3. **Scale Components**: Run multiple replicas with proper replication factors
4. **Add TLS**: Enable HTTPS for all endpoints
5. **Configure Retention**: Set appropriate data retention policies
6. **Set Resource Limits**: Configure CPU and memory limits
7. **Enable Monitoring**: Monitor the observability stack itself
8. **Backup Configuration**: Regular backups of Grafana dashboards and configurations

## License

This configuration is provided as-is for observability setup purposes.

## Support

For issues with specific components:
- [Grafana Documentation](https://grafana.com/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/)
- [Tempo Documentation](https://grafana.com/docs/tempo/)
- [Mimir Documentation](https://grafana.com/docs/mimir/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
