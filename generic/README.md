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
  type: ClusterIP
  port: 80

  # Optional: multiple named ports
  # ports:
  #   - name: http
  #     port: 80
  #     protocol: TCP
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

## ServiceMonitor

```yaml
serviceMonitor:
  enabled: true
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | Affinity for pod assignment |
| args | list | `[]` | The arguments to pass at runtime |
| autoscaling.behavior | object | `{"scaleDown":{},"scaleUp":{}}` | HPA behavior configuration |
| autoscaling.behavior.scaleDown | object | `{}` | Scale down behavior configuration |
| autoscaling.behavior.scaleUp | object | `{}` | Scale up behavior configuration |
| autoscaling.enabled | bool | `false` | Whether to enable autoscaling |
| autoscaling.maxReplicas | int | `100` | The maximum number of pods |
| autoscaling.minReplicas | int | `1` | The minimum number of pods |
| autoscaling.targetCPUUtilizationPercentage | int | `80` | The metrics to use for autoscaling |
| command | list | `[]` | The command to pass at runtime |
| configmap | object | `{"annotations":{},"binaryData":{},"data":{},"enabled":false,"labels":{},"name":""}` | ConfigMap configuration |
| configmap.annotations | object | `{}` | Additional annotations for the ConfigMap |
| configmap.binaryData | object | `{}` | ConfigMap binaryData |
| configmap.data | object | `{}` | ConfigMap data |
| configmap.enabled | bool | `false` | Whether to create a ConfigMap |
| configmap.labels | object | `{}` | Additional labels for the ConfigMap |
| configmap.name | string | `""` | ConfigMap name override (defaults to <fullname>) |
| cronjob.annotations | object | `{}` | CronJob annotations |
| cronjob.concurrencyPolicy | string | `"Forbid"` | How to treat concurrent executions (Allow|Forbid|Replace) |
| cronjob.enabled | bool | `false` | Enable creation of a Kubernetes CronJob |
| cronjob.failedJobsHistoryLimit | int | `1` | How many failed jobs should be kept |
| cronjob.job.activeDeadlineSeconds | string | `nil` | Optional overall deadline for the Job (seconds) |
| cronjob.job.backoffLimit | int | `6` | Number of retries before marking as failed |
| cronjob.job.completions | string | `nil` | Desired number of successfully finished pods |
| cronjob.job.parallelism | string | `nil` | Max number of pods running in parallel |
| cronjob.job.podAnnotations | object | `{}` | Additional annotations for the CronJob job pod template (defaults to podAnnotations) |
| cronjob.job.podLabels | object | `{}` | Additional labels for the CronJob job pod template (defaults to podLabels) |
| cronjob.job.restartPolicy | string | `"OnFailure"` | Pod restart policy for the CronJob job pod |
| cronjob.job.ttlSecondsAfterFinished | string | `nil` | Optional TTL to clean up finished Jobs (seconds) |
| cronjob.job.volumeMounts | list | `[]` | Additional volumeMounts for the CronJob job pod template (merged with top-level volumeMounts) |
| cronjob.job.volumes | list | `[]` | Additional volumes for the CronJob job pod template (merged with top-level volumes) |
| cronjob.labels | object | `{}` | CronJob labels |
| cronjob.schedule | string | `"0 * * * *"` | Cron schedule in standard Cron format |
| cronjob.startingDeadlineSeconds | string | `nil` | Optional deadline in seconds for starting the job if it misses scheduled time |
| cronjob.successfulJobsHistoryLimit | int | `3` | How many completed jobs should be kept |
| cronjob.suspend | bool | `false` | Whether to suspend subsequent executions |
| cronjob.timeZone | string | `""` | Optional timezone for the schedule (Kubernetes 1.27+) |
| deploymentAnnotations | object | `{}` | Annotations to add to deployments |
| envVars | object | `{}` |  |
| extraContainers | list | `[]` | Extra containers to run alongside the main container (sidecar containers) |
| extraObjects | list | `[]` | Extra Kubernetes objects to deploy |
| fullnameOverride | string | `""` | The full release name override |
| image.pullPolicy | string | `"IfNotPresent"` | The image pull policy |
| image.repository | string | `"nginx"` | The image repository |
| image.tag | string | `""` | Overrides the image tag whose default is the chart appVersion. |
| imagePullSecrets | list | `[]` | The secrets used to pull the image |
| ingress.annotations | object | `{}` | Ingress Annotations |
| ingress.className | string | `""` | Ingress Class Name |
| ingress.enabled | bool | `false` | Enable Ingress |
| ingress.hosts[0].host | string | `"chart-example.local"` |  |
| ingress.hosts[0].paths[0].path | string | `"/"` |  |
| ingress.hosts[0].paths[0].pathType | string | `"ImplementationSpecific"` |  |
| ingress.tls | list | `[]` | TLS configuration |
| job.activeDeadlineSeconds | string | `nil` | Optional overall deadline for the Job (seconds) |
| job.annotations | object | `{}` | Job annotations |
| job.backoffLimit | int | `6` | Number of retries before marking as failed |
| job.completions | string | `nil` | Desired number of successfully finished pods |
| job.enabled | bool | `false` | Enable creation of a Kubernetes Job |
| job.labels | object | `{}` | Job labels |
| job.parallelism | string | `nil` | Max number of pods running in parallel |
| job.podAnnotations | object | `{}` | Additional annotations for the Job pod template (defaults to podAnnotations) |
| job.podLabels | object | `{}` | Additional labels for the Job pod template (defaults to podLabels) |
| job.restartPolicy | string | `"OnFailure"` | Pod restart policy for the Job pod |
| job.ttlSecondsAfterFinished | string | `nil` | Optional TTL to clean up finished Jobs (seconds) |
| job.volumeMounts | list | `[]` | Additional volumeMounts for the Job pod template (merged with top-level volumeMounts) |
| job.volumes | list | `[]` | Additional volumes for the Job pod template (merged with top-level volumes) |
| lifecycle | object | `{"preStop":{"exec":{"command":["sh","-c","sleep 15 && kill -3 1"]}}}` | Lifecycle hooks |
| livenessProbe | object | `{"httpGet":{"path":"/_/health","port":"http"}}` | Live and Readiness Probes |
| nameOverride | string | `""` | The release name override |
| nodeSelector | object | `{}` | Node selector labels |
| persistence | object | `{"accessModes":["ReadWriteMany"],"claimName":"","enabled":false,"existingClaim":"","mountPath":"/data/app","name":"app-data","size":"1Gi","storageClassName":"do-block-storage","subPath":""}` | Persistence configuration for the main workload (Deployment/StatefulSet).  When enabled: - Deployment: creates a PVC (unless existingClaim is set) and mounts it. - StatefulSet: uses volumeClaimTemplates (unless existingClaim is set). |
| persistence.accessModes | list | `["ReadWriteMany"]` | Access modes for the PVC/volumeClaimTemplates. |
| persistence.claimName | Deployment only | `""` | PVC name to create + mount. When empty, defaults to: <release>-<chart>-data (i.e. <fullname>-data). |
| persistence.enabled | bool | `false` | Enable persistence (PVC/volumeClaimTemplates) for the main workload. |
| persistence.existingClaim | string | `""` | Use an existing PVC instead of creating one / using volumeClaimTemplates. |
| persistence.mountPath | string | `"/data/app"` | Path where the volume should be mounted. |
| persistence.name | string | `"app-data"` | The volume name (also the volumeClaimTemplates name for StatefulSets). |
| persistence.size | string | `"1Gi"` | Requested storage size. |
| persistence.storageClassName | string | `"do-block-storage"` | StorageClass name to use (empty means use the cluster default). |
| persistence.subPath | string | `""` | Optional subPath within the volume. |
| podAnnotations | object | `{}` | Annotations to add to the pods |
| podLabels | object | `{}` | Label to add to the pods |
| podSecurityContext | object | `{}` | The Pod Security Context |
| readinessProbe.httpGet.path | string | `"/_/health"` |  |
| readinessProbe.httpGet.port | string | `"http"` |  |
| replicaCount | int | `1` | Replica count |
| resources | object | `{}` | The Resources |
| secrets | list | `[]` | Secrets configuration |
| securityContext | object | `{}` | The Security Context |
| service.port | int | `80` | The service port |
| service.sessionAffinity.enabled | bool | `false` | Whether to enable session affinity |
| service.sessionAffinity.timeoutSeconds | int | `10800` | The session affinity timeout in seconds |
| service.sessionAffinity.type | string | `"ClientIP"` | The session affinity type |
| service.type | string | `"ClusterIP"` | The service type |
| serviceAccount.annotations | object | `{}` | Annotations to add to the service account |
| serviceAccount.automount | bool | `true` | Automatically mount a ServiceAccount's API credentials? |
| serviceAccount.create | bool | `true` | Specifies whether a service account should be created |
| serviceAccount.name | string | `""` | The name of the service account to use. |
| serviceMonitor | object | `{"annotations":{},"enabled":false,"endpoints":[]}` | ServiceMonitor configuration for Prometheus monitoring |
| serviceMonitor.annotations | object | `{}` | ServiceMonitor annotations |
| serviceMonitor.enabled | bool | `false` | Whether to enable ServiceMonitor |
| serviceMonitor.endpoints | list | `[]` | ServiceMonitor endpoints configuration |
| tolerations | list | `[]` | Tolerations for pod assignment |
| type | string | `"deployment"` | The type of workload to deploy. Can be 'deployment' or 'statefulset'. |
| volumeMounts | list | `[]` | Additional volumeMounts on the output Deployment definition. |
| volumes | list | `[]` | Additional volumes on the output Deployment definition. |
| vpa | object | `{"annotations":{},"enabled":false,"labels":{},"recommenders":[],"resourcePolicy":{},"updatePolicy":{"updateMode":"Off"}}` | Vertical Pod Autoscaler configuration  Note: VPA requires the VPA controller + CRDs installed in the cluster. |
| vpa.annotations | object | `{}` | Optional annotations for the VPA object |
| vpa.enabled | bool | `false` | Whether to enable VPA |
| vpa.labels | object | `{}` | Optional labels for the VPA object |
| vpa.recommenders | list | `[]` | Optional recommenders list |
| vpa.resourcePolicy | object | `{}` | Resource policy configuration Example: resourcePolicy:   containerPolicies:     - containerName: "*"       controlledResources: ["cpu", "memory"] |
| vpa.updatePolicy | object | `{"updateMode":"Off"}` | Update policy configuration Typical modes: Off | Initial | Recreate | Auto |
