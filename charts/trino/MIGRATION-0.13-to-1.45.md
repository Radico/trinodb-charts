# Migrating values from the 0.13 fork to 1.45.0+simon

The 0.13 fork (Trino 406) and this chart (upstream 1.45.0 plus Simon patches,
Trino 482) take differently shaped values. Most fork-only keys now have an
upstream equivalent, usually as a list where the fork used a map. This file maps
every key the production and staging values files use, then shows
`values-production.yaml` and `values-production-blue.yaml` converted for 482.

## Before deploying

- Install into an empty release. The Deployment selectors changed from
  `app`/`release`/`component` to the `app.kubernetes.io/*` labels, and selectors
  are immutable, so `helm upgrade` of a release created by the 0.13 chart fails.
  Blue/green already works this way: uninstall the old color, install the new one.
- Set `fullnameOverride` to the release name in each color file. 0.13 dropped
  the `-trino` suffix when the release name *contained* `trino`; 1.45 drops it
  only when the release name *equals* `trino`. Without the override every
  resource is renamed (`trino-blue` becomes `trino-blue-trino`).
- Set `server.autoscaling.targetMemoryUtilizationPercentage: ""`. Upstream
  defaults it to 80, and with `-Xms` above the memory request the HPA would sit
  at `maxReplicas`.

## Key mapping

| 0.13 fork key | 1.45.0+simon key and shape | Notes |
|---|---|---|
| `image.repository`, `image.pullPolicy` | same | |
| `image.tag` | same | `482-jmx` |
| `service.type` | same | |
| `serviceAccount.create`, `.name`, `.annotations` | same | IRSA role annotation passes through unchanged |
| `env` | same | |
| `coordinator.hostNetwork`, `worker.hostNetwork` | removed | Only ever `false`, which is the Kubernetes default |
| `coordinator.jvm.maxHeapSize`, `worker.jvm.maxHeapSize` | same | |
| `coordinator.additionalJVMConfig` (map), `worker.additionalJVMConfig` (map) | `<role>.jvm.minHeapSize` for `-Xms`, `<role>.additionalJVMConfig` (list) for everything else | Map keys (`one`, `ten`) are dropped, list order is render order |
| `coordinator.config.memory.heapHeadroomPerNode`, `worker.config.memory.heapHeadroomPerNode` | same | |
| `server.config.query.maxMemoryPerNode` | `coordinator.config.query.maxMemoryPerNode` and `worker.config.query.maxMemoryPerNode` | Set both to keep one value cluster-wide |
| `server.config.query.maxMemory` | same | Renders `query.max-memory`, so drop the duplicate line from `additionalConfigProperties` |
| `<role>.additionalPropertyFiles.access-control` | `accessControl: {type: properties, properties: ...}` | Rendered on coordinator and workers, as before |
| `<role>.additionalPropertyFiles.event-listener` | `eventListenerProperties` (list) | Rendered on coordinator and workers, as before |
| `<role>.additionalPropertyFiles.resource-groups` | `resourceGroups: {type: properties, properties: ...}` | Coordinator only. Workers never read it. `/etc/resource-groups.json` still ships in the image |
| `<role>.additionalPropertyFiles.session-property-config` | `sessionProperties: {type: properties, properties: ...}` | Coordinator only. `/etc/session-property-config.json` still ships in the image |
| `coordinator.additionalPropertyFiles.cluster-info` | `coordinator.additionalConfigFiles."cluster-info.properties"` | Key is the full file name. Content goes through `tpl` |
| `coordinator.annotations`, `worker.annotations` | same | Pod annotations. Values go through `tpl`; the Datadog `%%host%%` templates are unaffected |
| `coordinator.tolerations`, `worker.tolerations` | same (list) | |
| `coordinator.resources`, `worker.resources` | same | |
| `coordinator.affinity`, `worker.affinity` | same | Pod anti-affinity matches `release In [trino]`, which no release name matches; unchanged from 0.13 |
| `worker.livenessProbe.failureThreshold` (staging) | same | 1.45 adds a startup probe and switches readiness to `/usr/lib/trino/bin/health-check` |
| `server.workers`, `server.node.environment` | same | |
| `server.config.jmx.enabled` | `worker.additionalExposedPorts.jmx: {name: jmx, port: 9080, servicePort: 9080, protocol: TCP}` | Not upstream `jmx.enabled`: that sets its own RMI ports and defaults the registry to 9080, which the JMX exporter javaagent already uses |
| `jmx.rmiregistry.port=9070` in `additionalConfigProperties` | unchanged, list item | Datadog JMX port |
| `server.exchangeManager.useVolume`, `.pvcName` | removed | Use `<role>.additionalVolumes` and `<role>.additionalVolumeMounts` if a PVC is ever needed |
| `server.exchangeManager.baseDir` (comma-separated string) | `server.exchangeManager.baseDir` (list) | Upstream default is `{}` and renders no exchange manager, so set `name: filesystem` explicitly. Production used the 0.13 default `/tmp/trino-local-file-system-exchange-manager` |
| `server.autoscaling.enabled`, `.maxReplicas`, `.targetCPUUtilizationPercentage`, `.behavior` | same | |
| `server.autoscaling.metrics` | same | Simon patch, upstream lacks it |
| (none) | `server.autoscaling.targetMemoryUtilizationPercentage: ""` | See above |
| `additionalConfigProperties` (map) | `additionalConfigProperties` (list) | Map keys (`a1`, `a12a`) are dropped |
| `additionalExchangeManagerProperties` (map) | `additionalExchangeManagerProperties` (list) | Large and staging |
| `additionalCatalogs` | same (map of file name to content) | |
| `ingress.enabled`, `.className`, `.annotations`, `.hosts` | same | |
| Pod labels `app`, `release`, `component` (hardcoded in 0.13) | `coordinator.labels` and `worker.labels`, with `release: "{{ .Release.Name }}"` | Prometheus scrape relabeling, prometheus-adapter and pod-cost-manager select on these |
| Worker preStop and `terminationGracePeriodSeconds: 120` (hardcoded in 0.13) | `worker.lifecycle.preStop` and `worker.terminationGracePeriodSeconds: 120` | Not `worker.gracefulShutdown`: it requires file-based access control on workers and fails to render with `accessControl.type: properties` |
| Worker `fsGroup: 1000` (hardcoded in 0.13) | `securityContext.fsGroup: 1000` | Now applies to the coordinator too |
| tpch-only catalogs (0.13 default) | `catalogs.tpcds: null` | 1.45 adds tpcds by default |
| `imagePullSecrets: [registry-credentials]` (0.13 default) | dropped (1.45 default `[]`) | Set `imagePullSecrets` if the nodes cannot pull from ECR with their own role |

## Trino 482 configuration changes

| 406 setting | 482 setting |
|---|---|
| `hive.s3.sse.enabled=true`, `hive.s3.sse.type=S3`, `hive.s3.max-connections=1000` | `fs.native-s3.enabled=true`, `s3.sse.type=S3`, `s3.max-connections=1000` |
| `hive.allow-drop-table=true` | removed |
| `query.hash-partition-count=50` | `query.max-hash-partition-count=50` |
| `optimizer.use-mark-distinct=false` | `optimizer.distinct-aggregations-strategy=SINGLE_STEP` |
| `exchange.compression-enabled=true` | `exchange.compression-codec=LZ4` |
| `spill-compression-enabled=true` | `spill-compression-codec=LZ4` |
| `query.low-memory-killer.delay=5m` | removed |
| `join-distribution-type=PARTITIONED` | `join-distribution-type=AUTOMATIC` |
| `discovery-server.enabled=true` (rendered by the 0.13 chart) | removed |
| (none) | `experimental.thread-per-driver-scheduler-enabled=false`, keeps the timesharing executor so the waiting-splits autoscaling metric stays comparable |
| (none) | `deprecated.legacy-varchar-to-char-coercion=true` |
| (none) | JVM `-Dfile.encoding=UTF-8` |
| `protocol.v1.alternate-header-name=Presto` | unchanged |

## Converted `values-production.yaml`

Account IDs, the certificate ID and the Slack webhook are placeholders here;
keep the real values from the kubernetes repo.

```yaml
# Small/Lean/Fast Trino Cluster Configuration
# Smaller workers (54G heap) with higher concurrency for throughput-oriented workloads
# Chart 1.45.0+simon, Trino 482
# Use with: helm upgrade --install trino-blue ... -f values-production.yaml -f values-production-blue.yaml --version 1.45.0+simon

image:
  repository: "<account-id>.dkr.ecr.us-east-1.amazonaws.com/trino"
  tag: "482-jmx"
  pullPolicy: "Always"
service:
  type: "NodePort"
serviceAccount:
  create: true

env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name

# The 0.13 chart hardcoded fsGroup on workers
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000

# Keep only the tpch catalog, as the 0.13 chart rendered
catalogs:
  tpcds: null

# Applies to coordinator and workers
accessControl:
  type: properties
  properties: |
    access-control.name=trino-auth

eventListenerProperties:
  - event-listener.name=trino-events

# Coordinator only. The JSON files ship in the image at /etc/
resourceGroups:
  type: properties
  properties: |
    resource-groups.configuration-manager=file
    resource-groups.config-file=/etc/resource-groups.json

sessionProperties:
  type: properties
  properties: |
    session-property-config.configuration-manager=file
    session-property-manager.config-file=/etc/session-property-config.json

# System Resources - Smaller sizing for lean cluster
coordinator:
  # app/release/component are what Prometheus relabeling, prometheus-adapter and pod-cost-manager select on
  labels:
    app: trino
    release: "{{ .Release.Name }}"
    component: coordinator
  config:
    memory:
      heapHeadroomPerNode: "10GB"
    query:
      maxMemoryPerNode: "46GB"
  jvm:
    maxHeapSize: "96G"
    minHeapSize: "96G"
  additionalJVMConfig:
    - "-javaagent:/home/trino/jmx_prometheus_javaagent.jar=9080:/home/trino/config.yaml"
    - "-Dfile.encoding=UTF-8"
  additionalConfigFiles:
    cluster-info.properties: |-
      presto.cluster.name=trino
      notifications.slack.webhook="<slack-webhook-url>"
  annotations:
    karpenter.sh/do-not-disrupt: "true"
    ad.datadoghq.com/trino-coordinator.check_names: '["trino"]'
    ad.datadoghq.com/trino-coordinator.init_configs: '[{"is_jmx": true, "collect_default_metrics":  true, "service": "trino-coordinator"}]'
    ad.datadoghq.com/trino-coordinator.instances: '[{"host": "%%host%%","port":"9070"}]'
    ad.datadoghq.com/trino-coordinator.logs: >-
        [{
          "service": "trino-coordinator",
          "source": "java"
        }]
  tolerations:
  - key: "simondata.co/trino-coordinator-jmx"
    operator: "Exists"
    effect: "NoSchedule"
  resources:
    requests:
      cpu: 14
      memory: "110Gi"
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: release
                  operator: In
                  values:
                    - trino
                - key: app
                  operator: In
                  values:
                    - trino
            topologyKey: kubernetes.io/hostname
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: billing.simondata.io/product
            operator: In
            values:
            - presto
          - key: node.kubernetes.io/lifecycle
            operator: In
            values:
              - normal

worker:
  labels:
    app: trino
    release: "{{ .Release.Name }}"
    component: worker
  config:
    memory:
      heapHeadroomPerNode: "8GB"
    query:
      maxMemoryPerNode: "46GB"
  jvm:
    maxHeapSize: "54G"
    minHeapSize: "54G"
  additionalJVMConfig:
    - "-javaagent:/home/trino/jmx_prometheus_javaagent.jar=9080:/home/trino/config.yaml"
    - "-Dfile.encoding=UTF-8"
  # JMX exporter javaagent port, scraped by Prometheus on the pod IP
  additionalExposedPorts:
    jmx:
      name: jmx
      port: 9080
      servicePort: 9080
      protocol: TCP
  # Drain the worker before the pod goes away. Not worker.gracefulShutdown: that
  # needs file-based access control on workers and we run trino-auth.
  terminationGracePeriodSeconds: 120
  lifecycle:
    preStop:
      exec:
        command:
          - "/bin/sh"
          - "-c"
          - "/usr/bin/curl -v -X PUT -H \"X-Trino-User: root\" -d '\"SHUTTING_DOWN\"' -H \"Content-type: application/json\" http://localhost:8080/v1/info/state && /usr/bin/sleep 120"
  annotations:
    karpenter.sh/do-not-disrupt: "true"
    ad.datadoghq.com/trino-worker.check_names: '["trino"]'
    ad.datadoghq.com/trino-worker.init_configs: '[{"is_jmx": true, "collect_default_metrics":  true, "service": "trino-worker"}]'
    ad.datadoghq.com/trino-worker.instances: '[{"host": "%%host%%","port":"9070"}]'
    ad.datadoghq.com/trino-worker.logs: >-
        [{
          "service": "trino-worker",
          "source": "java"
        }]
  tolerations:
  - key: "simondata.co/trino-jmx"
    operator: "Exists"
    effect: "NoSchedule"
  resources:
    requests:
      cpu: 5
      memory: "50Gi"
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: release
                  operator: In
                  values:
                    - trino
                - key: app
                  operator: In
                  values:
                    - trino
            topologyKey: kubernetes.io/hostname
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: billing.simondata.io/product
            operator: In
            values:
            - presto
          - key: node.kubernetes.io/lifecycle
            operator: In
            values:
              - normal

server:
  workers: 2
  config:
    query:
      maxMemory: "100TB"
  # The 0.13 chart always rendered this default
  exchangeManager:
    name: filesystem
    baseDir:
      - /tmp/trino-local-file-system-exchange-manager
  autoscaling:
    enabled: true
    maxReplicas: 70
    targetCPUUtilizationPercentage: 80
    # Upstream defaults to 80. -Xms pins the heap above the memory request, so a memory target would hold the HPA at max.
    targetMemoryUtilizationPercentage: ""
    behavior:
      scaleDown:
        stabilizationWindowSeconds: 300
        policies:
        - type: Percent
          value: 10
          periodSeconds: 60
        - type: Pods
          value: 5
          periodSeconds: 60
        selectPolicy: Min
      scaleUp:
        stabilizationWindowSeconds: 120
        policies:
        - type: Percent
          value: 80
          periodSeconds: 120
        - type: Percent
          value: 40
          periodSeconds: 60
        - type: Percent
          value: 30
          periodSeconds: 60
        selectPolicy: Max

additionalConfigProperties:
  - protocol.v1.alternate-header-name=Presto
  - jmx.rmiregistry.port=9070
  # Spill configuration - enable spilling to disk for memory-intensive operators
  - query-max-spill-per-node=40GB
  - max-spill-per-node=50GB
  - spiller-threads=5
  - spill-enabled=true
  - spiller-spill-path=/tmp/trino-spill
  # Trigger spilling earlier - default 0.9 waits too long and HashAggregationOperator can OOM before revocation kicks in
  - memory-revoking-threshold=0.75
  - memory-revoking-target=0.5
  # Replaces optimizer.use-mark-distinct=false: MarkDistinctOperator can't spill and causes per-node OOM on COUNT(DISTINCT)
  - optimizer.distinct-aggregations-strategy=SINGLE_STEP
  - join-distribution-type=AUTOMATIC
  - redistribute-writes=true
  # query.max-memory is rendered by the chart from server.config.query.maxMemory
  - query.max-total-memory=200TB
  - query.max-execution-time=300m
  - query.max-stage-count=200
  - query.low-memory-killer.policy=total-reservation-on-blocked-nodes
  - retry-policy=QUERY
  - query.max-history=500
  - query.min-expire-age=30m
  - spill-compression-codec=LZ4
  - spill-encryption-enabled=true
  - task.info-update-interval=1s
  - http-server.process-forwarded=true
  - query.max-hash-partition-count=50
  - exchange.compression-codec=LZ4
  # Higher concurrency settings for smaller workers
  - task.max-worker-threads=32
  - task.concurrency=32
  - node-scheduler.max-splits-per-node=128
  - node-scheduler.max-pending-splits-per-task=32
  # Keep the timesharing executor so the waiting-splits autoscaling metric stays comparable to 406
  - experimental.thread-per-driver-scheduler-enabled=false
  - deprecated.legacy-varchar-to-char-coercion=true

additionalCatalogs:
  hive: |-
    connector.name=hive
    hive.metastore=glue
    hive.collect-column-statistics-on-write=true
    hive.compression-codec=NONE
    hive.parquet.use-column-names=true
    hive.recursive-directories=true
    fs.native-s3.enabled=true
    s3.sse.type=S3
    s3.max-connections=1000
    parquet.ignore-statistics=true

ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/tags: 'billing:product=presto'
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:us-east-1:<account-id>:certificate/<certificate-id>"
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '10'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '3'
    alb.ingress.kubernetes.io/healthcheck-path: '/v1/info'
    alb.ingress.kubernetes.io/target-node-labels: karpenter.sh/nodepool=trino-coordinator-jmx-normal-arm64
```

## Converted `values-production-blue.yaml`

Only `fullnameOverride` is new. Green, blue-large and green-large take the same
line with their own release name.

```yaml
# Keep the 0.13 resource names (trino-blue, trino-blue-worker, ...). Chart 1.45
# only drops the chart-name suffix when the release name equals "trino" exactly.
fullnameOverride: "trino-blue"

serviceAccount:
  name: "trino-blue-k8s-production"
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::<account-id>:role/eks/simon/platform/production/trino-blue-k8s-production"

server:
  workers: 2
  node:
    environment: "production"
  autoscaling:
    enabled: true
    maxReplicas: 70
    metrics:
      - type: Object
        object:
          metric:
            name: trino_execution_executor_TaskExecutor_WaitingSplits_blue_total
          describedObject:
            apiVersion: v1
            kind: Namespace
            name: trino
          target:
            type: Value
            value: "3000"  # Increased from 25 to reduce sensitivity
    behavior:
      scaleUp:
        stabilizationWindowSeconds: 60  # Reduced from default 120 for faster scale-up
        policies:
        - type: Percent
          value: 100  # Increased from 80% for aggressive scale-up
          periodSeconds: 60
        - type: Pods
          value: 10  # Allow adding 10 pods at once
          periodSeconds: 60
        selectPolicy: Max  # Use the policy that scales up most
      scaleDown:
        stabilizationWindowSeconds: 180  # Reduced from 300 — buffer exporter protects draining workers
        policies:
        - type: Percent
          value: 25
          periodSeconds: 60
        - type: Pods
          value: 8
          periodSeconds: 60
        selectPolicy: Min  # Use the policy that scales down least

ingress:
  hosts:
    - host: "trino-blue.production.simondata.co"
      paths:
        - path: /
          pathType: Prefix
```
