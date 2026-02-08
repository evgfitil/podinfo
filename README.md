# podinfo - Helm Chart Exercise

## Exercise Goal

You have a working Go microservice (podinfo) deployed to Kubernetes using plain YAML manifests. Your task is to create a production-quality Helm chart that templatizes these manifests, making the deployment configurable and reusable.

The `manifests/` directory contains all the Kubernetes resources you need to convert into Helm templates. Use the application reference below to decide which values should be parameterized in your chart.

## Application Reference

### Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 9898 | HTTP | Primary API and Web UI |
| 9999 | gRPC | gRPC API |
| 9797 | HTTP | Prometheus metrics (`/metrics`) |
| 9899 | HTTPS | TLS-enabled HTTP (optional, requires cert) |

### Health Probes

| Probe | Endpoint | Port |
|-------|----------|------|
| Liveness | `/healthz` | 9898 (HTTP) |
| Readiness | `/readyz` | 9898 (HTTP) |

### Environment Variables

All environment variables use the `PODINFO_` prefix. CLI flag names are uppercased and dashes become underscores (e.g., `--cache-server` becomes `PODINFO_CACHE_SERVER`).

Key variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `PODINFO_PORT` | `9898` | HTTP listen port |
| `PODINFO_LEVEL` | `info` | Log level (debug, info, warn, error, fatal, panic) |
| `PODINFO_CACHE_SERVER` | (empty) | Redis address: `tcp://<host>:<port>` |
| `PODINFO_UI_COLOR` | `#34577c` | Web UI background color |
| `PODINFO_UI_MESSAGE` | `greetings from podinfo v<version>` | Web UI greeting message |
| `PODINFO_BACKEND_URL` | (empty) | Backend service URL for `/echo` endpoint |

### Volumes

| Mount Path | Type | Description |
|------------|------|-------------|
| `/data` | emptyDir | Application data directory |
| `/data/cert` | Secret | TLS certificate and key (optional) |

### Container

- Base image: `alpine:3.23`
- Runs as non-root user `app`
- Binary path: `/podinfo`

### Web API

- `GET /` -- runtime information (JSON)
- `GET /version` -- version and git commit hash
- `GET /metrics` -- Prometheus metrics
- `GET /healthz` -- liveness probe endpoint
- `GET /readyz` -- readiness probe endpoint
- `POST /readyz/enable` -- mark instance ready
- `POST /readyz/disable` -- mark instance not ready
- `GET /env` -- environment variables (JSON)
- `GET /headers` -- request headers (JSON)
- `GET /delay/{seconds}` -- artificial latency
- `GET /status/{code}` -- return specific HTTP status code
- `GET /panic` -- crash the process (exit 255)
- `POST /echo` -- forward to backend and echo content
- `POST /cache/{key}` -- store value in Redis
- `GET /cache/{key}` -- retrieve value from Redis
- `DELETE /cache/{key}` -- delete key from Redis

### gRPC API

Served on port 9999:

- `/grpc.health.v1.Health/Check` -- health checking
- `/grpc.EchoService/Echo` -- echo content
- `/grpc.VersionService/Version` -- version info
- `/grpc.DelayService/Delay` -- artificial latency
- `/grpc.EnvService/Env` -- environment variables
- `/grpc.InfoService/Info` -- runtime information

## Project Structure

```text
cmd/podinfo/       Application entrypoint
cmd/podcli/        CLI client (used in health checks)
pkg/               Application packages (API, version, signals, fscache)
ui/                Web UI (vue.html)
manifests/         Plain Kubernetes manifests (your starting point)
charts/podinfo/    Reference Helm chart (for mentors)
Dockerfile         Container image build
Makefile           Build and test commands
```

## Kubernetes Manifests

The `manifests/` directory contains plain YAML files for all resources you need to templatize:

| File | Resource | What to Parameterize |
|------|----------|---------------------|
| `deployment.yaml` | Deployment | image, replicas, resources, probes, env vars, ports |
| `service.yaml` | Service | port names and numbers, selector labels |
| `hpa.yaml` | HorizontalPodAutoscaler | min/max replicas, CPU target, enable/disable |
| `ingress.yaml` | Ingress | host, paths, TLS, enable/disable |
| `serviceaccount.yaml` | ServiceAccount | name, annotations, enable/disable |
| `pdb.yaml` | PodDisruptionBudget | minAvailable/maxUnavailable, enable/disable |
| `redis-config.yaml` | ConfigMap | Redis configuration (maxmemory, eviction policy) |
| `redis-deployment.yaml` | Deployment | image, resources, probes, enable/disable |
| `redis-service.yaml` | Service | port, selector labels |

Apply manifests directly to verify they work:

```bash
kubectl apply -f manifests/
```

## Redis Dependency

Podinfo uses Redis as an optional cache backend. The `manifests/` directory includes three Redis resources:

- `redis-config.yaml` -- ConfigMap with `redis.conf` (64 MB memory limit, allkeys-lru eviction, persistence disabled)
- `redis-deployment.yaml` -- single-replica Deployment running `redis:8.4.0` with liveness (TCP) and readiness (`redis-cli ping`) probes
- `redis-service.yaml` -- ClusterIP Service exposing port 6379

The podinfo Deployment connects to Redis via the `--cache-server` flag:

```text
--cache-server=tcp://podinfo-redis:6379
```

This flag can also be set through the `PODINFO_CACHE_SERVER` environment variable. When Redis is available, the `/cache/{key}` API endpoints (GET, POST, DELETE) become functional.

When creating your Helm chart, consider making Redis an optional dependency that can be enabled or disabled through chart values.

## Reference Helm Chart

The `charts/podinfo/` directory contains a full production Helm chart. This is provided as a reference for mentors and should not be modified during the exercise.

## Makefile Targets

| Target | Description |
|--------|-------------|
| `make run` | Run the application locally (HTTP on :9898, gRPC on :9999) |
| `make test` | Run unit tests with coverage |
| `make build` | Build `podinfo` and `podcli` binaries |
| `make tidy` | Clean and update Go module dependencies |
| `make vet` | Run `go vet` on all packages |
| `make fmt` | Format Go source code |
| `make build-charts` | Lint and package Helm charts |
| `make build-container` | Build Docker container image |

## Quick Start

Build and run locally:

```bash
make build
make run
# Open http://localhost:9898
```

Run tests:

```bash
make test
```

Build container image:

```bash
make build-container
```
