# Nginx Helm Chart

A Helm chart for deploying nginx on Kubernetes, built to demonstrate Helm best practices. Uses a custom nginx image that serves colour-coded pages at `/app/red` and `/app/yellow`.

## Table of Contents

- [Architecture](#architecture)
- [Design Decisions](#design-decisions)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Docker Image](#docker-image)

## Architecture

```
.
├── Chart.yaml                  # Chart metadata and version
├── values.yaml                 # Default configuration values
├── templates/
│   ├── _helpers.tpl            # Reusable template functions
│   ├── NOTES.txt               # Post-install instructions
│   ├── configmap.yaml          # Nginx server configuration
│   ├── deployment.yaml         # Application deployment
│   ├── hpa.yaml                # Horizontal Pod Autoscaler
│   ├── ingress.yaml            # Ingress resource
│   ├── namespace.yaml          # Optional namespace creation
│   ├── pdb.yaml                # Pod Disruption Budget
│   ├── service.yaml            # ClusterIP/LoadBalancer service
│   └── serviceaccount.yaml     # Dedicated service account
├── docker/
│   ├── Dockerfile              # Multi-stage build on Ubuntu 22.04
│   └── default.conf            # Nginx server config for image
└── .github/workflows/
    ├── lint.yml                # Helm lint + template validation
    └── docker.yml              # Docker build validation
```

## Design Decisions

### Why `_helpers.tpl` for Naming and Labels?

Every resource in this chart uses `{{ include "nginx.fullname" . }}` for its name and `{{ include "nginx.labels" . }}` for labels, defined once in `_helpers.tpl`. Without this:

- Resource names are fragile — changing the naming convention means editing every template file.
- Multiple installations in the same namespace collide because names are hardcoded.
- Labels are inconsistent across resources, breaking label-based selection and monitoring.

The helper pattern means `helm install staging ./` and `helm install production ./` both work in the same namespace without conflicts, because `fullname` incorporates the release name.

### Why `app.kubernetes.io/*` Labels Instead of Custom Labels?

Kubernetes has a [recommended label convention](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/). Using `app.kubernetes.io/name`, `app.kubernetes.io/instance`, and `app.kubernetes.io/version` means:

- Tools like `kubectl`, Lens, and ArgoCD can identify and group resources automatically.
- Prometheus service discovery works out of the box with standard label selectors.
- Other teams looking at your cluster understand the labels without reading your chart source.

### Why ConfigMap for Nginx Config Instead of Baking into the Image?

The original chart baked the nginx config into the Docker image via `COPY default.conf`. This chart mounts it from a ConfigMap instead:

- **Config changes do not require a new image build.** Adjusting a header, adding a location block, or changing the listen port is a `helm upgrade` — not a Docker build, push, and redeploy cycle.
- **The deployment checksum annotation** (`checksum/config`) triggers a rolling restart when the ConfigMap changes, so pods always pick up the latest config without manual intervention.
- **Environment-specific config** is possible by overriding values per environment rather than building separate images.

### Why Health Probes?

The deployment includes both liveness and readiness probes hitting `/healthz`:

- **Readiness probes** tell Kubernetes when a pod is ready to receive traffic. Without them, the Service sends traffic to pods that are still starting, causing errors.
- **Liveness probes** detect deadlocked processes. If nginx hangs, Kubernetes restarts the container rather than leaving it in a broken state.
- **The `/healthz` endpoint** is a dedicated lightweight route that returns 200 without touching the application logic or generating access logs.

### Why Resource Requests and Limits?

Every container has explicit CPU and memory requests/limits:

- **Requests** inform the scheduler. Without them, Kubernetes has no idea how much capacity the pod needs and may schedule it on a node that cannot support it.
- **Limits** prevent a single pod from consuming all node resources. A memory leak in one pod should not crash neighbouring pods.
- **HPA depends on requests.** The Horizontal Pod Autoscaler calculates utilisation as `current usage / requested resources`. Without requests, HPA cannot function.

### Why a Service Account per Release?

Each Helm release creates its own ServiceAccount rather than using `default`:

- **Least privilege**: If the application later needs AWS access (via IRSA) or GCP Workload Identity, the service account is already in place — just add annotations.
- **Audit trail**: Pod identity in audit logs shows which service account was used, making it easy to trace actions back to a specific release.
- **RBAC scoping**: RoleBindings can target the specific service account rather than granting permissions to every pod in the namespace.

### Why Pod Disruption Budget?

The PDB ensures a minimum number of pods remain available during voluntary disruptions (node drains, cluster upgrades, spot evictions):

- Without a PDB, `kubectl drain` can terminate all pods simultaneously, causing downtime.
- `minAvailable: 1` guarantees at least one pod is always serving traffic during rolling updates or node maintenance.
- PDBs only restrict voluntary disruptions — involuntary disruptions (OOM kills, node failures) are not affected.

### Why Optional Ingress Instead of Always-On?

The Ingress resource is disabled by default (`ingress.enabled: false`) because:

- Not every cluster has an Ingress controller installed. Creating an Ingress resource without a controller just produces an orphaned resource.
- Some deployments only need internal access via ClusterIP.
- The template supports `ingressClassName`, annotations, and TLS — enabling it is a single values override.

### Why Multi-Stage Docker Build?

The Dockerfile uses a multi-stage build:

- **Build stage** installs nginx and prepares the static content and configuration. This layer includes `apt-get` caches and build artefacts.
- **Final stage** starts from a clean Ubuntu 22.04 image and copies only the built content and config. The final image does not carry leftover `apt` cache or build-time dependencies.
- The result is a smaller, cleaner image with a reduced attack surface.

## Getting Started

### Prerequisites

- [Helm](https://helm.sh/docs/intro/install/) v3+
- A Kubernetes cluster (local or remote)

### Install

```bash
helm install my-nginx .
```

### Upgrade

```bash
helm upgrade my-nginx .
```

### Install with Custom Values

```bash
helm install my-nginx . \
  --set replicaCount=3 \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=myapp.example.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].pathType=Prefix
```

### Uninstall

```bash
helm uninstall my-nginx
```

## Configuration

| Parameter | Description | Default |
|---|---|---|
| `replicaCount` | Number of replicas | `2` |
| `image.repository` | Container image repository | `nginx` |
| `image.tag` | Container image tag | `""` (uses `appVersion`) |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `containerPort` | Container port | `80` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `80` |
| `ingress.enabled` | Enable ingress | `false` |
| `ingress.className` | Ingress class name | `""` |
| `ingress.hosts` | Ingress host rules | `[{host: nginx.example.com}]` |
| `resources.requests.cpu` | CPU request | `100m` |
| `resources.requests.memory` | Memory request | `64Mi` |
| `resources.limits.cpu` | CPU limit | `200m` |
| `resources.limits.memory` | Memory limit | `128Mi` |
| `autoscaling.enabled` | Enable HPA | `false` |
| `autoscaling.minReplicas` | Minimum replicas for HPA | `2` |
| `autoscaling.maxReplicas` | Maximum replicas for HPA | `10` |
| `autoscaling.targetCPUUtilizationPercentage` | Target CPU utilisation | `80` |
| `podDisruptionBudget.enabled` | Enable PDB | `false` |
| `podDisruptionBudget.minAvailable` | Minimum available pods | `1` |
| `serviceAccount.create` | Create service account | `true` |
| `namespace.create` | Create namespace resource | `false` |

## Docker Image

### Build

```bash
cd docker
docker build --platform=linux/amd64 -t nginx:1.0.0 .
```

### Routes

| Path | Description |
|---|---|
| `/app/red` | Red themed page |
| `/app/yellow` | Yellow themed page |
| `/healthz` | Health check endpoint |
