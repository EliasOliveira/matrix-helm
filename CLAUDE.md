# CLAUDE.md

This file provides guidance for AI assistants working with the matrix-helm repository.

## Repository Overview

This is a collection of Kubernetes Helm charts managed by **ArgoCD**. It contains infrastructure services (Kafka, Elasticsearch, Bitwarden) and custom microservices, all deployed to Kubernetes clusters.

## Directory Structure

```
matrix-helm/
├── helm/                          # Base Helm chart (Guestbook template/scaffold)
├── bitwarden/                     # Bitwarden password manager deployment
├── elasticsearch/                 # ELK Stack Elasticsearch chart
├── kafka/
│   ├── broker/helm/              # Kafka broker + Zookeeper (StatefulSets)
│   └── kafka-ui/                 # Kafka UI management web interface
└── micro-services/
    ├── node-sandbox/             # Node.js sandbox application
    ├── firebase-connector/       # Firebase connector (Kafka consumer/producer)
    ├── ftp-server/              # FTP server service
    ├── kafka-stream-java/       # Kafka Streams processor (Java)
    └── kafka-stream-node/       # Kafka Streams processor (Node.js)
```

## Chart Structure Convention

Every chart follows the same layout:

```
<chart>/
├── Chart.yaml              # Chart metadata (name, version, appVersion)
├── values.yaml             # Default configuration values
├── values-production.yaml  # Production overrides (typically: service.type: LoadBalancer)
└── templates/
    ├── _helpers.tpl        # Template helpers (name, fullname, chart label)
    ├── deployment.yaml     # Kubernetes Deployment
    ├── service.yaml        # Kubernetes Service
    ├── config.yaml         # ConfigMap (microservices only)
    └── NOTES.txt           # Post-install usage notes
```

Exceptions:
- `kafka/broker/helm/templates/` contains `kafka-broker.yaml`, `zookeeper.yaml`, `rbac.yaml`, `kafka-config.yaml`, and `kafka-cli.yaml` instead of the standard layout.
- `bitwarden/` has a more complex template structure for its multi-component architecture.

## Key Conventions

### Naming

All helper templates use the `helm-guestbook.*` naming scheme (inherited from the scaffold). Chart names are truncated to 63 characters for DNS compliance. The helpers define:
- `helm-guestbook.name` — chart name with optional `nameOverride`
- `helm-guestbook.fullname` — fully qualified name with release prefix
- `helm-guestbook.chart` — chart name + version label

### Labels

Deployments use four standard labels:
```yaml
app: {{ template "helm-guestbook.name" . }}
chart: {{ template "helm-guestbook.chart" . }}
release: {{ .Release.Name }}
heritage: {{ .Release.Service }}
```

### Versioning

Charts use semantic versioning with release candidate suffixes:
- Chart version: `0.1.0-rc7`, `0.1.1-rc3`, etc.
- App version: `1.0.0-rc3-<short-sha>-<date>` or `0.0.1-SNAPSHOT-<short-sha>`

### Service Types

- **Development** (default): `ClusterIP` for internal-only access
- **Production** (`values-production.yaml`): `LoadBalancer` for external access
- **Kafka broker**: `NodePort` (default values)
- **StatefulSets** (Kafka, Zookeeper): Headless services (`clusterIP: None`)

### Image Repositories

Custom microservice images are published to Docker Hub under the `eliasjunioroliveira/` namespace. Infrastructure images use upstream sources (elastic, solsson, ibmcom, provectuslabs, ghcr.io/bitwarden).

## Key Internal Service Endpoints

These DNS names are referenced across charts for inter-service communication:

| Service | Endpoint |
|---------|----------|
| Kafka broker | `kafka-broker.default.svc.cluster.local:9092` |
| Zookeeper | `zookeeper-svc.default.svc.cluster.local:2181` |
| Elasticsearch | `elasticsearch:9200` |
| Kafka UI | `kafka-ui:8080` |

## Deployment Model

- All charts are deployed via **ArgoCD** (GitOps — push to repo triggers sync).
- No CI/CD pipelines (GitHub Actions, Makefile, etc.) exist in this repository.
- No automated Helm tests or validation scripts are present.

## Common Helm Commands

```bash
# List all releases
helm list -aq --namespace default

# Delete a release
helm delete <release-name> --namespace default

# Template a chart locally (dry-run)
helm template <release-name> ./<chart-path>

# Install/upgrade a chart
helm upgrade --install <release-name> ./<chart-path> -f ./<chart-path>/values.yaml
```

## Working with This Repository

### Adding a New Microservice

1. Copy an existing microservice chart directory (e.g., `micro-services/node-sandbox/`).
2. Update `Chart.yaml` with the new chart name, version, and appVersion.
3. Update `values.yaml` with the correct image repository, tag, and port.
4. Update `templates/config.yaml` with service-specific configuration.
5. Optionally set `nameOverride` in `values.yaml`.
6. Create `values-production.yaml` with `service.type: LoadBalancer` if external access is needed.

### Modifying an Existing Chart

- Configuration changes go in `values.yaml` (defaults) or `values-production.yaml` (production overrides).
- Template logic changes go in the corresponding file under `templates/`.
- Bump the chart version in `Chart.yaml` when making changes.

### Storage and Persistence

- Kafka and Zookeeper use `hostPath` PersistentVolumes (5Gi each).
- Bitwarden requires a `shared-storage` StorageClass with `ReadWriteMany` access mode.
- Most microservices are stateless and do not require persistent storage.

### Secrets

- **Bitwarden**: Requires a pre-created `custom-secret` Kubernetes secret.
- **Firebase connector**: Requires a `firebase` secret with `FIREBASE_SERVICE_ACCOUNT` key.
- Secrets are **not** managed in this repository — they must be created manually or via an external secrets manager.

## Git Workflow

- **Primary branch**: `master`
- **Development branch**: `develop`
- Commit messages follow the pattern: `Create <component-name>` for new charts.
- No branch protection, PR templates, or automated checks are configured.
