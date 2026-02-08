# Podinfo

Podinfo is a tiny web application made with Go that showcases best practices of running microservices in Kubernetes.

Podinfo is used by CNCF projects like [Flux](https://github.com/fluxcd/flux2) and [Flagger](https://github.com/fluxcd/flagger) for end-to-end testing and workshops.

Container image: `ghcr.io/stefanprodan/podinfo:6.10.1`

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

## Kubernetes Manifests

The `manifests/` directory contains plain YAML files for deploying podinfo and Redis:

| File | Resource | Key Configuration |
|------|----------|-------------------|
| `deployment.yaml` | Deployment | image, replicas, resources, probes, env vars, ports |
| `service.yaml` | Service | port names and numbers, selector labels |
| `hpa.yaml` | HorizontalPodAutoscaler | min/max replicas, CPU target |
| `ingress.yaml` | Ingress | host, paths, TLS |
| `serviceaccount.yaml` | ServiceAccount | name, annotations |
| `pdb.yaml` | PodDisruptionBudget | minAvailable/maxUnavailable |
| `redis-config.yaml` | ConfigMap | Redis configuration (maxmemory, eviction policy) |
| `redis-deployment.yaml` | Deployment | image, resources, probes |
| `redis-service.yaml` | Service | port, selector labels |

Deploy with plain manifests:

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

## Quick Start

Deploy to a Kubernetes cluster:

```bash
kubectl create namespace podinfo
kubectl apply -f manifests/ -n podinfo
kubectl port-forward -n podinfo svc/podinfo 9898:9898
# Open http://localhost:9898
```
