# Helm Charts

A collection of Helm charts for deploying applications on Kubernetes.

## Available Charts

| Chart | Version | Description |
|-------|---------|-------------|
| [generic](./generic/) | 1.0.9 | A generic Helm chart for deploying containerized applications with support for both Deployment and StatefulSet workloads |
| [raw](./raw/) | 0.2.5 | A raw chart to deploy loose kubernetes resources |

## Prerequisites

Before using these Helm charts, you should have the following prerequisites:

- Access to Kubernetes cluster (If needed contact your friendly neighbourhood DevOps engineer)
- Helm >= v3.14.3
- (**Optional**) helmfile >= v0.162.0 to install these charts

## Installation

### Using Helmfile (Recommended)

To install a Helm chart, the easiest way is to create a `helmfile.yaml` with needed values and run:

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

You can also install charts directly using Helm:

```bash
# Add the repository (if hosted)
helm repo add txlabs https://your-chart-repo-url

# Install a chart
helm install my-release txlabs/generic -f values.yaml
```

Or install from local directory:

```bash
helm install my-release ./generic -f values.yaml
```

## Chart Documentation

Each chart has its own README with detailed documentation:

- **[generic](./generic/README.md)** - Full-featured chart supporting Deployments, StatefulSets, Jobs, CronJobs, Services, Ingress, HPA, VPA, ServiceMonitor, and more
- **[atlas-ip-updater](./atlas-ip-updater/README.md)** - CronJob for MongoDB Atlas IP whitelist management
- **[raw](./raw/README.md)** - Deploy raw Kubernetes resources

## Generic Chart Features

The `generic` chart is a versatile chart that supports:

### Workload Types
- **Deployment** - Standard Kubernetes Deployment
- **StatefulSet** - For stateful applications

### Additional Resources
- **Job** - One-time batch jobs
- **CronJob** - Scheduled jobs
- **Service** - ClusterIP, NodePort, LoadBalancer
- **Ingress** - HTTP/HTTPS routing
- **ConfigMap** - Configuration data
- **Secrets** - Sensitive data
- **PersistentVolumeClaim** - Storage

### Autoscaling
- **HPA** - Horizontal Pod Autoscaler
- **VPA** - Vertical Pod Autoscaler

### Monitoring
- **ServiceMonitor** - Prometheus monitoring

### Example Usage

#### Basic Deployment

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

#### StatefulSet with Persistence

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

#### Deployment with Ingress

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

#### CronJob

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

#### HPA Configuration

```yaml
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
```

## Development

### Generating Chart Documentation

This repository uses [helm-docs](https://github.com/norwoodj/helm-docs) to generate chart documentation. The `README.md.gotmpl` template is used to generate individual chart READMEs.

To regenerate documentation:

```bash
helm-docs
```

### Pre-commit Hooks

This repository uses pre-commit hooks. Install them with:

```bash
pre-commit install
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run `helm lint` on your chart
5. Submit a pull request

## License

See individual chart directories for license information.
