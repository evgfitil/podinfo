# Prepare podinfo student branch for Helm chart exercise

## Overview

Prepare a clean branch from the podinfo fork for a Kubernetes training exercise. Students receive a working Go microservice with plain K8s manifests and must create a full production-quality Helm chart from scratch. The existing production Helm chart stays in the repo as a reference for mentors.

The result is a self-contained repository branch where:

- the application builds and runs locally and in a container;
- plain Kubernetes manifests cover all resources students must templatize into a Helm chart (Deployment, Service, HPA, Ingress, ServiceAccount, PDB);
- README documents application ports, probes, env vars and other details needed for chart parameterization;
- mentors can reference the production Helm chart in `charts/podinfo/` for grading/guidance;
- everything unrelated to the exercise is removed to reduce noise.

## Context (from discovery)

**Repository:** podinfo v6.10.1 — Go microservice template for Kubernetes (HTTP + gRPC APIs)

**Application details (critical for chart creation):**

- **Ports:** HTTP 9898 (primary), gRPC 9999, metrics 9797, HTTPS 9899 (optional)
- **Probes:** liveness `/healthz`, readiness `/readyz` on HTTP port
- **Env vars:** prefix `PODINFO_`, dashes become underscores (e.g. `PODINFO_CACHE_SERVER`)
- **Volumes:** `/data` (emptyDir), `/data/cert` (TLS secret, optional)
- **Container:** runs as non-root user `app`, base `alpine:3.23`
- **Redis:** optional cache dependency (`--cache-server=tcp://host:port`)

**Current structure (30+ top-level items):**

- `cmd/`, `pkg/`, `ui/` — application source code
- `Dockerfile`, `Dockerfile.base`, `Dockerfile.xx` — container builds
- `charts/podinfo/` — full production Helm chart (29 files, 7 resource types)
- `kustomize/` — bare-bones K8s manifests (Deployment, Service, HPA only)
- `deploy/` — complex Kustomize overlays (57 files)
- `timoni/` — CUE-based deployment module (150+ files)
- `.github/`, `.cosign/`, `.notation/` — CI/CD and image signing
- `otel/`, `test/` — dev/CI tooling
- `.goreleaser.yml`, `cloudbuild.yaml` — release automation

**Gap analysis:** kustomize/ contains only 3 resource types. The Helm chart has 7+ (Deployment, Service, HPA, Ingress, ServiceAccount, PDB, ServiceMonitor, Certificate, Redis subchart, hooks). Students need plain YAML for at least the core 6 resource types to build a full chart.

## Development Approach

- No code changes to the application itself
- File/directory deletions, Makefile cleanup, new manifests from `helm template`, README rewrite
- Work in a dedicated branch `student-exercise`
- Verify the app still builds and tests pass after cleanup

## Implementation Steps

### Task 1: Create branch and remove unnecessary directories

- [x] create branch `student-exercise` from `master`
- [x] remove `.github/` (CI/CD workflows, dependabot, actions)
- [x] remove `.cosign/` (cosign image signing config)
- [x] remove `.notation/` (notation image signing config)
- [x] remove `timoni/` (CUE-based deployment module, 150+ files)
- [x] remove `otel/` (OpenTelemetry docker-compose dev setup)
- [x] remove `test/` (E2E shell scripts for CI)
- [x] remove `deploy/` (complex Kustomize overlays: 57 files)
- [x] verify: `make test` passes

### Task 2: Remove unnecessary root files

- [x] remove `Dockerfile.base` (base image build)
- [x] remove `Dockerfile.xx` (cross-platform buildx)
- [x] remove `.goreleaser.yml` (release automation for podcli)
- [x] remove `cloudbuild.yaml` (GCP Cloud Build pipeline)
- [x] remove `CLAUDE.md` (project-specific Claude instructions)
- [x] verify: `make build` succeeds
- [x] verify: `make build-container` succeeds

### Task 3: Simplify Makefile

Remove targets that reference deleted components or are irrelevant for the exercise.

- [x] remove targets: `build-xx`, `build-base`, `push-base`, `test-container`, `push-container`, `version-set`, `release`, `swagger`, `timoni-build`
- [x] keep targets: `run`, `test`, `build`, `tidy`, `vet`, `fmt`, `build-charts`, `build-container`
- [x] verify: `make test` passes

### Task 4: Generate full set of Kubernetes manifests

Replace bare-bones `kustomize/` with a complete `manifests/` directory containing all resource types students must templatize.

- [x] run `helm template` on `charts/podinfo/` with Ingress, HPA, ServiceAccount, PDB enabled to generate rendered manifests
- [x] create `manifests/` directory with individual plain YAML files, cleaned of Helm-specific labels/annotations:
  - `deployment.yaml` — with probes, resources, securityContext, volumes, all ports
  - `service.yaml` — with named ports (http, grpc, metrics)
  - `hpa.yaml` — with CPU metric
  - `ingress.yaml` — basic Ingress resource
  - `serviceaccount.yaml` — ServiceAccount
  - `pdb.yaml` — PodDisruptionBudget
- [x] remove `kustomize/` directory (replaced by `manifests/`)
- [x] verify manifests are valid: `kubectl apply --dry-run=client -f manifests/`

### Task 5: Update README.md

Rewrite README for the student exercise context. Include application reference information so students know what to parameterize in their Helm chart.

- [ ] describe the exercise purpose and goal
- [ ] document application details:
  - ports: HTTP 9898, gRPC 9999, metrics 9797, HTTPS 9899
  - health probes: `/healthz` (liveness), `/readyz` (readiness)
  - key env vars: `PODINFO_` prefix, important ones (PORT, LEVEL, CACHE_SERVER, UI_COLOR, UI_MESSAGE, BACKEND_URL)
  - volumes: `/data` (emptyDir), `/data/cert` (TLS, optional)
  - container user: `app` (non-root)
- [ ] document available Makefile targets (only the kept ones)
- [ ] explain `manifests/` directory — plain K8s resources as the basis for Helm chart
- [ ] note that `charts/podinfo/` is a reference Helm chart (for mentors)
- [ ] remove all references to deleted components

### Task 6: Verify build and unit tests

- [ ] verify `make test` passes (unit tests)
- [ ] verify `make build` succeeds (binary builds)
- [ ] verify `make build-container` succeeds (Docker image)
- [ ] verify `make run` starts the application and responds on `localhost:9898/healthz`
- [ ] verify `make build-charts` works (Helm lint on reference chart)
- [ ] verify no broken references to deleted files/directories

### Task 7: Integration test — Helm vs manifests deployment comparison

Deploy the application into two namespaces (Helm install vs plain manifests), compare that the application behaves identically.

**Prerequisites:** Kind, Helm, kubectl, Docker

**IMPORTANT: use a local kubeconfig only.** Create `kubeconfig` in the project root directory and use it for all kubectl/helm commands in this task. Do NOT use the default `~/.kube/config` or any other existing clusters/contexts.

```
export KUBECONFIG=$(pwd)/kubeconfig
```

All kubectl and helm commands below assume this env var is set.

**Setup:**

- [ ] set `KUBECONFIG=$(pwd)/kubeconfig`
- [ ] create Kind cluster: `kind create cluster --name podinfo-test --kubeconfig $(pwd)/kubeconfig`
- [ ] install metrics-server (required for HPA):
  ```
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  kubectl patch deployment metrics-server -n kube-system \
    --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
  kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=90s
  ```
- [ ] build Docker image: `docker build -t podinfo:test .`
- [ ] load image into Kind: `kind load docker-image podinfo:test --name podinfo-test`

**Namespace `test-helm` — deploy via Helm:**

- [ ] install with matching config:
  ```
  helm install podinfo charts/podinfo/ \
    --namespace test-helm --create-namespace \
    --set image.repository=podinfo \
    --set image.tag=test \
    --set image.pullPolicy=Never \
    --set serviceAccount.enabled=true \
    --set ingress.enabled=true \
    --set hpa.enabled=true \
    --set pdb.minAvailable=1
  ```
- [ ] wait for pods ready: `kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=podinfo -n test-helm --timeout=60s`

**Namespace `test-manifests` — deploy via plain manifests:**

- [ ] create namespace: `kubectl create namespace test-manifests`
- [ ] apply all manifests with image override (manifests have hardcoded image, replace for test):
  ```
  sed 's|ghcr.io/stefanprodan/podinfo:.*|podinfo:test|' manifests/deployment.yaml | kubectl apply -n test-manifests -f -
  kubectl apply -f manifests/service.yaml -n test-manifests
  kubectl apply -f manifests/serviceaccount.yaml -n test-manifests
  kubectl apply -f manifests/ingress.yaml -n test-manifests
  kubectl apply -f manifests/pdb.yaml -n test-manifests
  kubectl apply -f manifests/hpa.yaml -n test-manifests
  ```
- [ ] wait for pods ready: `kubectl wait --for=condition=ready pod -l app=podinfo -n test-manifests --timeout=60s`

**Compare application behavior:**

- [ ] port-forward both:
  ```
  kubectl port-forward -n test-helm svc/podinfo 9801:9898 &
  kubectl port-forward -n test-manifests svc/podinfo 9802:9898 &
  ```
- [ ] compare `/version` — must return same version in both
- [ ] compare `/healthz` — both return HTTP 200
- [ ] compare `/readyz` — both return HTTP 200
- [ ] compare `/` — both return JSON with same structure (ignore `hostname` field, it differs per pod)
- [ ] compare Service ports — both expose http (9898) and grpc (9999)

**Compare Kubernetes resources:**

- [ ] compare HPA in both namespaces — same MINPODS, MAXPODS, TARGETS:
  ```
  kubectl get hpa -n test-helm
  kubectl get hpa -n test-manifests
  ```
- [ ] verify PDB exists in both: `kubectl get pdb -n test-helm && kubectl get pdb -n test-manifests`
- [ ] verify ServiceAccount exists in both: `kubectl get sa -n test-helm && kubectl get sa -n test-manifests`
- [ ] verify Ingress exists in both: `kubectl get ingress -n test-helm && kubectl get ingress -n test-manifests`

### Task 8: Commit all changes

- [ ] commit all changes to `student-exercise` branch

**ralphex: this is the last automated task. Stop execution after this task is complete.**

### Task 9: Manual review of deployments (performed by reviewer, not by ralphex)

> **This task is performed manually by the reviewer.** ralphex must NOT execute this task.

Both deployments remain running in the Kind cluster after Task 7.

Port-forward commands for access:

```
kubectl port-forward -n test-helm svc/podinfo 9801:9898 &
kubectl port-forward -n test-manifests svc/podinfo 9802:9898 &
```

- `http://localhost:9801` — Helm deployment
- `http://localhost:9802` — manifests deployment

Useful commands for manual inspection:

```
kubectl get all -n test-helm
kubectl get all -n test-manifests
kubectl describe deployment -n test-helm
kubectl describe deployment -n test-manifests
kubectl get hpa,pdb,ingress,sa -n test-helm
kubectl get hpa,pdb,ingress,sa -n test-manifests
```

Checklist for the reviewer:

- [ ] verify both apps respond identically on `/`, `/version`, `/healthz`, `/readyz`
- [ ] inspect Kubernetes resources in both namespaces (Deployment, Service, HPA, PDB, Ingress, ServiceAccount)
- [ ] review the committed branch for completeness and cleanliness
- [ ] cleanup when done: `kind delete cluster --name podinfo-test`

## Expected final structure

```
podinfo/
├── cmd/
│   ├── podinfo/             # main server application
│   └── podcli/              # CLI client (used in health probes)
├── pkg/
│   ├── api/
│   │   ├── http/            # HTTP API (25+ endpoints, tests)
│   │   └── grpc/            # gRPC API (9 services, tests)
│   ├── signals/             # graceful shutdown
│   ├── fscache/             # config file watcher
│   └── version/             # version constants
├── ui/                      # vue.html web interface
├── charts/
│   └── podinfo/             # REFERENCE Helm chart (mentors only)
├── manifests/               # plain K8s manifests for students
│   ├── deployment.yaml      # Deployment with probes, resources, security
│   ├── service.yaml         # Service with named ports
│   ├── hpa.yaml             # HorizontalPodAutoscaler
│   ├── ingress.yaml         # Ingress
│   ├── serviceaccount.yaml  # ServiceAccount
│   └── pdb.yaml             # PodDisruptionBudget
├── Dockerfile
├── Makefile                 # simplified
├── README.md                # exercise-oriented with app reference info
├── LICENSE
├── go.mod
├── go.sum
├── .gitignore
└── .gitattributes
```

## Post-Completion

**Manual verification:**

- mentor walks through the reference Helm chart to confirm it's useful as a grading reference
- student attempts the exercise: clones repo, builds the app, reviews manifests, creates Helm chart
- verify manifests in `manifests/` can be applied to a cluster: `kubectl apply -f manifests/`
