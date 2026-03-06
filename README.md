# Helm Charts

A collection of Helm charts for deploying applications on Kubernetes.

## Available Charts

| Chart | Version | Description |
|-------|---------|-------------|
| [generic](./generic/) | 1.0.9 | A generic Helm chart for deploying containerized applications with support for both Deployment and StatefulSet workloads |
| [raw](./raw/) | 0.2.5 | A raw chart to deploy loose Kubernetes resources |

## Prerequisites

- Access to Kubernetes cluster (If needed contact your friendly neighbourhood DevOps engineer)
- Helm >= v3.14.3
- (**Optional**) helmfile >= v0.162.0 to install these charts
- (**Optional**) pre-commit for git hooks

## Installation

### Using Helmfile (Recommended)

Create a `helmfile.yaml` with needed values and run:

```bash
helmfile template
helmfile apply
```

Or use helmfile only to generate resources and apply them with kubectl:

```bash
helmfile template | kubectl apply -f -
```

Verify that the chart is deployed successfully:

> Note: `kubectl` is a better suited tool for this

```bash
helmfile status
```

### Using Helm Directly

Install from local directory:

```bash
helm install my-release ./generic -f values.yaml
```

Or from a hosted repository:

```bash
helm repo add txlabs https://your-chart-repo-url
helm install my-release txlabs/generic -f values.yaml
```

## Common Commands

```bash
# Lint a chart
helm lint ./generic
helm lint ./raw

# Template render (preview output without deploying)
helm template my-release ./generic -f values.yaml
helm template my-release ./raw -f values.yaml
```

---

## Generic Chart

The `generic` chart is a versatile, all-in-one chart used to deploy all applications into Kubernetes via helmfile. A single `values.yaml` controls which resources are created.

For detailed documentation and all configuration options, see **[generic/README.md](./generic/README.md)**.

### Supported Resources

| Category | Resource | Enabled By |
|----------|----------|------------|
| **Workloads** | Deployment | `type: deployment` (default) |
| | StatefulSet | `type: statefulset` |
| | Job | `job.enabled: true` |
| | CronJob | `cronjob.enabled: true` |
| **Networking** | Service | `service.enabled: true` (default) |
| | Ingress | `ingress.enabled: true` |
| | NetworkPolicy | `networkPolicy.enabled: true` |
| **Configuration** | ConfigMap | `configmap.enabled: true` |
| | Secret | `secrets: [...]` |
| | ServiceAccount | `serviceAccount.create: true` (default) |
| **Storage** | PersistentVolumeClaim | `persistence.enabled: true` |
| **Scaling** | HorizontalPodAutoscaler | `autoscaling.enabled: true` |
| | VerticalPodAutoscaler | `vpa.enabled: true` |
| **Reliability** | PodDisruptionBudget | `podDisruptionBudget.enabled: true` |
| **Monitoring** | ServiceMonitor | `serviceMonitor.enabled: true` |
| **Extensibility** | Extra Objects | `extraObjects: [...]` |
| **Helm** | NOTES.txt | Always enabled — shows post-install access instructions |

### Pod-Level Features

| Feature | Configuration Key |
|---------|-------------------|
| Init containers | `initContainers` |
| Sidecar containers | `extraContainers` |
| Environment variables | `envVars`, `envFrom` |
| Health probes | `livenessProbe`, `readinessProbe`, `startupProbe` |
| Deployment strategy | `strategy` (Deployment), `updateStrategy` (StatefulSet) |
| Topology spread | `topologySpreadConstraints` |
| Graceful shutdown | `terminationGracePeriodSeconds` |
| DNS configuration | `dnsPolicy`, `dnsConfig` |
| Lifecycle hooks | `lifecycle` |
| Revision history | `revisionHistoryLimit` |
| Security contexts | `podSecurityContext`, `securityContext` |
| Scheduling | `nodeSelector`, `tolerations`, `affinity` |

### Example: Basic Deployment

```yaml
# helmfile.yaml
releases:
  - name: my-app
    chart: ./generic
    values:
      - image:
          repository: nginx
          tag: latest
        service:
          type: ClusterIP
          port: 80
```

### Example: StatefulSet with Persistence

```yaml
type: statefulset
persistence:
  enabled: true
  name: app-data
  mountPath: /data/app
  accessModes:
    - ReadWriteMany
  size: 1Gi
  storageClassName: do-block-storage
```

### Example: Deployment with Ingress

```yaml
type: deployment
service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: example.com
      paths:
        - path: /
          pathType: Prefix
```

### Example: Production-Ready Deployment

```yaml
type: deployment
replicaCount: 3

image:
  repository: my-app
  tag: v1.2.3

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 0

initContainers:
  - name: migrate
    image: my-app:v1.2.3
    command: ['./migrate']

envFrom:
  - secretRef:
      name: my-app-secrets

startupProbe:
  httpGet:
    path: /_/health
    port: http
  failureThreshold: 30
  periodSeconds: 10

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
```

### Example: CronJob

```yaml
cronjob:
  enabled: true
  schedule: "0 * * * *"
  job:
    volumes:
      - name: data
        persistentVolumeClaim:
          claimName: my-existing-pvc
    volumeMounts:
      - name: data
        mountPath: /data
```

---

## Raw Chart

The `raw` chart deploys arbitrary Kubernetes manifests without opinionated templates. Useful for one-off resources that don't fit the generic chart.

For detailed documentation, see **[raw/README.md](./raw/README.md)**.

- **`resources`** — Static YAML manifests deployed as-is (with standard labels merged in)
- **`templates`** — YAML manifests with Go template support (processed via `tpl`)

### Example: Raw Resources

```yaml
resources:
  - apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
      name: high-priority
    value: 1000000
    globalDefault: false
    description: "High priority workloads"
```

### Example: Raw Templates (with Go templating)

```yaml
templates:
  - |
    apiVersion: v1
    kind: Secret
    metadata:
      name: my-secret
    stringData:
      api-key: {{ .Values.apiKey }}
```

---

## Architecture

### Generic Chart Internals

- **`templates/_helpers.tpl`** — Defines template helpers (`generic.fullname`, `generic.labels`, `generic.selectorLabels`, `generic.serviceAccountName`, `generic.persistenceClaimName`, `generic.configmapName`). All resource templates reference these.
- **`values.yaml`** — Single values file controls all resource generation. Key top-level keys: `type`, `image`, `service`, `ingress`, `persistence`, `autoscaling`, `vpa`, `podDisruptionBudget`, `networkPolicy`, `configmap`, `secrets`, `job`, `cronjob`, `serviceMonitor`, `initContainers`, `extraContainers`, `extraObjects`.
- **Persistence behavior differs by workload type**: Deployments create a standalone PVC; StatefulSets use `volumeClaimTemplates`.
- **Jobs/CronJobs** do not render liveness/readiness probes (unlike Deployment/StatefulSet) since health probes are not appropriate for batch workloads.
- **`extraObjects`** allows injecting arbitrary Kubernetes manifests alongside the chart's managed resources.

### Raw Chart Internals

- Merges each item in `resources`/`templates` with a base metadata template (`raw.resource`) that adds standard labels (`app`, `chart`, `release`, `heritage`).
- `templates` entries support Go templating (processed via `tpl`), while `resources` entries are static YAML.
- The `raw/ci/` directory contains test value files used for CI validation.

## Development

### Pre-commit Hooks

This repository uses pre-commit hooks. Install them with:

```bash
pre-commit install
```

### Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run `helm lint` on your chart
5. Submit a pull request

## License

See individual chart directories for license information.
