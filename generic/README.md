# generic

![Version: 1.0.9](https://img.shields.io/badge/Version-1.0.9-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 1.0.0](https://img.shields.io/badge/AppVersion-1.0.0-informational?style=flat-square)

A generic Helm chart for deploying containerized applications with support for both Deployment and StatefulSet workloads

## Prerequisites

Before using this Helm chart, you should have the following prerequisites:

- Access to Kubernetes cluster (If needed contact your friendly neighbourhood DevOps engineer)
- Helm >= v3.14.3
- (**Optional**) helmfile >= v0.162.0 to install this chart

## Installation

> Note: **examples** can be found in the repository

To install this Helm chart, the easiest is to create a helmfile.yaml with needed values and run:

```
helmfile template
helmfile apply
```

Or use helmfile only to generate resources and apply them with kubectl like so:

```
helmfile template | kubectl -f -
```

Verify that the chart is deployed successfully:

> Note: `kubectl` is a better suited tool for this

```
helmfile status
```

## Persistence (Deployment/StatefulSet)

This chart can create and mount persistent storage for the main workload using `persistence.*`.

- Deployment: creates a `PersistentVolumeClaim` (unless `persistence.existingClaim` is set).
- StatefulSet: uses `volumeClaimTemplates` (unless `persistence.existingClaim` is set).

### Example (StatefulSet volumeClaimTemplates)

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

### Example (Deployment PVC with explicit name)

```yaml
type: deployment
persistence:
  enabled: true
  name: app-data
  mountPath: /data/app
  # PVC name to create + mount
  claimName: my-app-pvc
  size: 1Gi
  storageClassName: do-block-storage
```

## Filesystem mounts (Job/CronJob)

Jobs/CronJobs do not create PVCs automatically.

- You can mount an existing PVC by setting `persistence.enabled=true` and `persistence.existingClaim`.
- If your main workload is a Deployment and `persistence.enabled=true`, the chart will create the PVC and Jobs/CronJobs will mount it automatically.

You can also continue to reference PVCs manually via `job.volumes` / `cronjob.job.volumes`.

### CronJob + emptyDir

```yaml
cronjob:
  enabled: true
  schedule: "*/5 * * * *"
  job:
    volumes:
      - name: workdir
        emptyDir: {}
    volumeMounts:
      - name: workdir
        mountPath: /work
```

### CronJob + existing PVC

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

## ConfigMap

This chart can optionally create a ConfigMap via `configmap.*`. You can then mount it using `volumes` / `volumeMounts` (or reference it via `envVars`).

```yaml
configmap:
  enabled: true
  data:
    APP_ENV: production
    LOG_LEVEL: info

volumes:
  - name: app-config
    configMap:
      name: my-release-generic

volumeMounts:
  - name: app-config
    mountPath: /etc/config
    readOnly: true
```

## Secrets

This chart can optionally create Secrets via `secrets` (rendered as `stringData`).

```yaml
secrets:
  - name: my-app-secret
    items:
      - key: username
        value: admin
      - key: password
        value: supersecret
```

## Service

```yaml
service:
  enabled: true  # set to false for job-only deployments
  type: ClusterIP
  port: 80
  annotations: {}  # useful for AWS LB controller, external-dns, etc.

  # Optional: multiple named ports
  # ports:
  #   - name: http
  #     port: 80
  #     protocol: TCP

  # Optional: session affinity
  # sessionAffinity:
  #   enabled: true
  #   type: ClientIP
  #   timeoutSeconds: 10800
```

## Ingress

```yaml
ingress:
  enabled: true
  className: nginx
  annotations: {}
  hosts:
    - host: example.com
      paths:
        - path: /
          pathType: Prefix
          # Optional overrides per-path
          # serviceName: my-release-generic
          # servicePort: 80
  tls: []
  # - secretName: example-tls
  #   hosts:
  #     - example.com
```

## Init Containers

```yaml
initContainers:
  - name: init-db
    image: busybox:latest
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting; sleep 2; done']
```

## Sidecar Containers

```yaml
extraContainers:
  - name: log-shipper
    image: fluentd:latest
    volumeMounts:
      - name: shared-logs
        mountPath: /var/log/app
```

## Environment Variables

### Individual env vars

```yaml
envVars:
  - name: APP_ENV
    value: production
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: my-secret
        key: password
```

### Load from ConfigMap/Secret (envFrom)

```yaml
envFrom:
  - configMapRef:
      name: my-configmap
  - secretRef:
      name: my-secret
```

## Probes

```yaml
# Startup probe (for slow-starting apps)
startupProbe:
  httpGet:
    path: /_/health
    port: http
  failureThreshold: 30
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /_/health
    port: http

readinessProbe:
  httpGet:
    path: /_/health
    port: http
```

## Deployment Strategy

```yaml
# For Deployments
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%

# For StatefulSets
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 0
```

## HPA

### Example (CPU-based)

```yaml
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80
  # Optional tuning
  # behavior:
  #   scaleUp:
  #     stabilizationWindowSeconds: 0
  #     policies:
  #       - type: Percent
  #         value: 100
  #         periodSeconds: 15
  #   scaleDown:
  #     stabilizationWindowSeconds: 300
  #     policies:
  #       - type: Percent
  #         value: 20
  #         periodSeconds: 60
```

## VPA

VPA (Vertical Pod Autoscaler) adjusts pod resource requests (and optionally limits). It requires the VPA controller + CRDs installed.

### Example (recommendations only)

```yaml
vpa:
  enabled: true
  updatePolicy:
    updateMode: Off
  # resourcePolicy:
  #   containerPolicies:
  #     - containerName: "*"
  #       controlledResources: ["cpu", "memory"]
```

### Example (auto updates + bounds)

```yaml
vpa:
  enabled: true
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        controlledResources: ["cpu", "memory"]
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
```

## PodDisruptionBudget

Ensures minimum pod availability during voluntary disruptions (node drains, cluster upgrades).

```yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 1          # or use percentage: "50%"
  # maxUnavailable: 1      # alternative to minAvailable
```

## NetworkPolicy

Controls ingress/egress traffic to pods.

```yaml
networkPolicy:
  enabled: true
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              name: my-namespace
      ports:
        - protocol: TCP
          port: 80
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8
      ports:
        - protocol: TCP
          port: 443
```

## ServiceMonitor

```yaml
serviceMonitor:
  enabled: true
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

## Pod Scheduling

```yaml
# Topology spread (HA across zones/nodes)
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app

nodeSelector: {}
tolerations: []
affinity: {}
```

## Pod Settings

```yaml
terminationGracePeriodSeconds: 60
dnsPolicy: ClusterFirst
dnsConfig:
  nameservers:
    - 1.1.1.1

# Revision history (Deployment only)
revisionHistoryLimit: 10
```

## Extra Objects

Deploy arbitrary Kubernetes resources alongside the chart.

```yaml
extraObjects:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: my-extra-config
    data:
      key: value
```

## NOTES.txt (Post-Install Instructions)

The chart includes a `NOTES.txt` template that Helm automatically renders and displays in the terminal after `helm install` or `helm upgrade`. It provides access instructions based on your configuration:

- **Ingress enabled** — Shows the configured host URLs
- **NodePort service** — Shows commands to get the node IP and port
- **LoadBalancer service** — Shows command to watch for the external IP
- **ClusterIP service** (default) — Shows the internal cluster DNS address

Example output after install:

```
1. Application my-app has been deployed.

2. Access the application inside the cluster:
   my-app.default.svc.cluster.local:80
```

No configuration is needed — the notes are generated automatically from your `service` and `ingress` values.

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| type | string | `"deployment"` | The type of workload to deploy. Can be 'deployment' or 'statefulset'. |
| replicaCount | int | `1` | Replica count |
| image.repository | string | `"nginx"` | The image repository |
| image.pullPolicy | string | `"IfNotPresent"` | The image pull policy |
| image.tag | string | `""` | Overrides the image tag whose default is the chart appVersion. |
| imagePullSecrets | list | `[]` | The secrets used to pull the image |
| nameOverride | string | `""` | The release name override |
| fullnameOverride | string | `""` | The full release name override |
| serviceAccount.create | bool | `true` | Specifies whether a service account should be created |
| serviceAccount.automount | bool | `true` | Automatically mount a ServiceAccount's API credentials? |
| serviceAccount.annotations | object | `{}` | Annotations to add to the service account |
| serviceAccount.name | string | `""` | The name of the service account to use. |
| strategy | object | `{}` | Deployment strategy (only used when type=deployment) |
| updateStrategy | object | `{}` | StatefulSet update strategy (only used when type=statefulset) |
| revisionHistoryLimit | string | `""` | Number of old ReplicaSets to retain (only used when type=deployment) |
| deploymentAnnotations | object | `{}` | Annotations to add to deployments |
| podAnnotations | object | `{}` | Annotations to add to the pods |
| podLabels | object | `{}` | Labels to add to the pods |
| podSecurityContext | object | `{}` | The Pod Security Context |
| securityContext | object | `{}` | The Security Context |
| service.enabled | bool | `true` | Whether to create a Service (set to false for job-only deployments) |
| service.type | string | `"ClusterIP"` | The service type |
| service.port | int | `80` | The service port |
| service.annotations | object | `{}` | Service annotations (useful for AWS LB controller, external-dns, etc.) |
| service.sessionAffinity.enabled | bool | `false` | Whether to enable session affinity |
| service.sessionAffinity.type | string | `"ClientIP"` | The session affinity type |
| service.sessionAffinity.timeoutSeconds | int | `10800` | The session affinity timeout in seconds |
| ingress.enabled | bool | `false` | Enable Ingress |
| ingress.className | string | `""` | Ingress Class Name |
| ingress.annotations | object | `{}` | Ingress Annotations |
| ingress.hosts | list | see values.yaml | Ingress hosts configuration |
| ingress.tls | list | `[]` | TLS configuration |
| command | list | `[]` | The command to pass at runtime |
| args | list | `[]` | The arguments to pass at runtime |
| envVars | object | `{}` | Environment variables |
| envFrom | list | `[]` | Environment variables from ConfigMaps or Secrets |
| resources | object | `{}` | The Resources |
| livenessProbe | object | `{"httpGet":{"path":"/_/health","port":"http"}}` | Liveness probe configuration |
| readinessProbe | object | `{"httpGet":{"path":"/_/health","port":"http"}}` | Readiness probe configuration |
| startupProbe | object | `{}` | Startup probe (useful for slow-starting containers) |
| podDisruptionBudget.enabled | bool | `false` | Whether to create a PodDisruptionBudget |
| podDisruptionBudget.annotations | object | `{}` | PDB annotations |
| podDisruptionBudget.minAvailable | string | `""` | Minimum number/percentage of pods that must remain available |
| podDisruptionBudget.maxUnavailable | string | `""` | Maximum number/percentage of pods that can be unavailable |
| networkPolicy.enabled | bool | `false` | Whether to create a NetworkPolicy |
| networkPolicy.annotations | object | `{}` | NetworkPolicy annotations |
| networkPolicy.policyTypes | list | `[]` | Policy types to enforce (Ingress, Egress, or both) |
| networkPolicy.ingress | list | `[]` | Ingress rules |
| networkPolicy.egress | list | `[]` | Egress rules |
| autoscaling.enabled | bool | `false` | Whether to enable autoscaling |
| autoscaling.minReplicas | int | `1` | The minimum number of pods |
| autoscaling.maxReplicas | int | `100` | The maximum number of pods |
| autoscaling.targetCPUUtilizationPercentage | int | `80` | Target CPU utilization for autoscaling |
| autoscaling.behavior | object | `{"scaleDown":{},"scaleUp":{}}` | HPA behavior configuration |
| autoscaling.behavior.scaleDown | object | `{}` | Scale down behavior configuration |
| autoscaling.behavior.scaleUp | object | `{}` | Scale up behavior configuration |
| vpa.enabled | bool | `false` | Whether to enable VPA |
| vpa.labels | object | `{}` | Optional labels for the VPA object |
| vpa.annotations | object | `{}` | Optional annotations for the VPA object |
| vpa.updatePolicy | object | `{"updateMode":"Off"}` | Update policy configuration (Off, Initial, Recreate, Auto) |
| vpa.resourcePolicy | object | `{}` | Resource policy configuration |
| vpa.recommenders | list | `[]` | Optional recommenders list |
| volumes | list | `[]` | Additional volumes on the output Deployment definition. |
| persistence.enabled | bool | `false` | Enable persistence (PVC/volumeClaimTemplates) for the main workload. |
| persistence.name | string | `"app-data"` | The volume name (also the volumeClaimTemplates name for StatefulSets). |
| persistence.mountPath | string | `"/data/app"` | Path where the volume should be mounted. |
| persistence.subPath | string | `""` | Optional subPath within the volume. |
| persistence.existingClaim | string | `""` | Use an existing PVC instead of creating one / using volumeClaimTemplates. |
| persistence.claimName | string | `""` | (Deployment only) PVC name to create + mount. Defaults to `<fullname>-data`. |
| persistence.accessModes | list | `["ReadWriteMany"]` | Access modes for the PVC/volumeClaimTemplates. |
| persistence.size | string | `"1Gi"` | Requested storage size. |
| persistence.storageClassName | string | `"do-block-storage"` | StorageClass name to use (empty means use the cluster default). |
| volumeMounts | list | `[]` | Additional volumeMounts on the output Deployment definition. |
| nodeSelector | object | `{}` | Node selector labels |
| tolerations | list | `[]` | Tolerations for pod assignment |
| affinity | object | `{}` | Affinity for pod assignment |
| topologySpreadConstraints | list | `[]` | Topology spread constraints for pod distribution across zones/nodes |
| terminationGracePeriodSeconds | string | `""` | Termination grace period in seconds |
| dnsPolicy | string | `""` | DNS policy for the pod (ClusterFirst, Default, ClusterFirstWithHostNet, None) |
| dnsConfig | object | `{}` | DNS config for the pod |
| lifecycle | object | `{"preStop":{"exec":{"command":["sh","-c","sleep 15 && kill -3 1"]}}}` | Lifecycle hooks |
| serviceMonitor.enabled | bool | `false` | Whether to enable ServiceMonitor |
| serviceMonitor.annotations | object | `{}` | ServiceMonitor annotations |
| serviceMonitor.endpoints | list | `[]` | ServiceMonitor endpoints configuration |
| secrets | list | `[]` | Secrets configuration |
| configmap.enabled | bool | `false` | Whether to create a ConfigMap |
| configmap.name | string | `""` | ConfigMap name override (defaults to `<fullname>`) |
| configmap.labels | object | `{}` | Additional labels for the ConfigMap |
| configmap.annotations | object | `{}` | Additional annotations for the ConfigMap |
| configmap.data | object | `{}` | ConfigMap data |
| configmap.binaryData | object | `{}` | ConfigMap binaryData |
| initContainers | list | `[]` | Init containers to run before the main container |
| extraContainers | list | `[]` | Extra containers to run alongside the main container (sidecar containers) |
| extraObjects | list | `[]` | Extra Kubernetes objects to deploy |
| job.enabled | bool | `false` | Enable creation of a Kubernetes Job |
| job.annotations | object | `{}` | Job annotations |
| job.labels | object | `{}` | Job labels |
| job.backoffLimit | int | `6` | Number of retries before marking as failed |
| job.completions | string | `nil` | Desired number of successfully finished pods |
| job.parallelism | string | `nil` | Max number of pods running in parallel |
| job.activeDeadlineSeconds | string | `nil` | Optional overall deadline for the Job (seconds) |
| job.ttlSecondsAfterFinished | string | `nil` | Optional TTL to clean up finished Jobs (seconds) |
| job.restartPolicy | string | `"OnFailure"` | Pod restart policy for the Job pod |
| job.podAnnotations | object | `{}` | Additional annotations for the Job pod template (defaults to podAnnotations) |
| job.podLabels | object | `{}` | Additional labels for the Job pod template (defaults to podLabels) |
| job.volumes | list | `[]` | Additional volumes for the Job pod template (merged with top-level volumes) |
| job.volumeMounts | list | `[]` | Additional volumeMounts for the Job pod template (merged with top-level volumeMounts) |
| cronjob.enabled | bool | `false` | Enable creation of a Kubernetes CronJob |
| cronjob.annotations | object | `{}` | CronJob annotations |
| cronjob.labels | object | `{}` | CronJob labels |
| cronjob.schedule | string | `"0 * * * *"` | Cron schedule in standard Cron format |
| cronjob.timeZone | string | `""` | Optional timezone for the schedule (Kubernetes 1.27+) |
| cronjob.concurrencyPolicy | string | `"Forbid"` | How to treat concurrent executions (Allow\|Forbid\|Replace) |
| cronjob.startingDeadlineSeconds | string | `nil` | Optional deadline in seconds for starting the job if it misses scheduled time |
| cronjob.suspend | bool | `false` | Whether to suspend subsequent executions |
| cronjob.successfulJobsHistoryLimit | int | `3` | How many completed jobs should be kept |
| cronjob.failedJobsHistoryLimit | int | `1` | How many failed jobs should be kept |
| cronjob.job.backoffLimit | int | `6` | Number of retries before marking as failed |
| cronjob.job.completions | string | `nil` | Desired number of successfully finished pods |
| cronjob.job.parallelism | string | `nil` | Max number of pods running in parallel |
| cronjob.job.activeDeadlineSeconds | string | `nil` | Optional overall deadline for the Job (seconds) |
| cronjob.job.ttlSecondsAfterFinished | string | `nil` | Optional TTL to clean up finished Jobs (seconds) |
| cronjob.job.restartPolicy | string | `"OnFailure"` | Pod restart policy for the CronJob job pod |
| cronjob.job.podAnnotations | object | `{}` | Additional annotations for the CronJob job pod template (defaults to podAnnotations) |
| cronjob.job.podLabels | object | `{}` | Additional labels for the CronJob job pod template (defaults to podLabels) |
| cronjob.job.volumes | list | `[]` | Additional volumes for the CronJob job pod template (merged with top-level volumes) |
| cronjob.job.volumeMounts | list | `[]` | Additional volumeMounts for the CronJob job pod template (merged with top-level volumeMounts) |
