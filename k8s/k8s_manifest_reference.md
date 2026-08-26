# Kubernetes YAML Manifest Reference

**Scope:** Exhaustive field-by-field reference for the most commonly authored Kubernetes resource kinds.
**API versions targeted:** the current stable (GA) API group/version for each kind, as of Kubernetes 1.30–1.33 (the stable window covering late 2025–2026 releases). Beta/alpha fields are called out explicitly where relevant.
**Conventions used below:**
- `# required` marks fields with no default that must be set.
- `# optional (default: X)` marks optional fields and their default.
- `# generated` marks fields the API server/controllers populate — don't set these yourself in manifests you write; they show up when you `kubectl get -o yaml` a live object.
- `<...>` is a placeholder; enums are shown as `A | B | C`.
- Nesting mirrors real YAML indentation.

---

## 1. Resource Index

| # | Kind | apiVersion | Category |
|---|------|-----------|----------|
| 1 | Pod | `v1` | Workload |
| 2 | Deployment | `apps/v1` | Workload |
| 3 | ReplicaSet | `apps/v1` | Workload |
| 4 | StatefulSet | `apps/v1` | Workload |
| 5 | DaemonSet | `apps/v1` | Workload |
| 6 | Job | `batch/v1` | Workload |
| 7 | CronJob | `batch/v1` | Workload |
| 8 | Service | `v1` | Networking |
| 9 | Ingress | `networking.k8s.io/v1` | Networking |
| 10 | NetworkPolicy | `networking.k8s.io/v1` | Networking |
| 11 | ConfigMap | `v1` | Config |
| 12 | Secret | `v1` | Config |
| 13 | PersistentVolume | `v1` | Storage |
| 14 | PersistentVolumeClaim | `v1` | Storage |
| 15 | StorageClass | `storage.k8s.io/v1` | Storage |
| 16 | Namespace | `v1` | Cluster admin |
| 17 | ServiceAccount | `v1` | RBAC |
| 18 | Role | `rbac.authorization.k8s.io/v1` | RBAC |
| 19 | RoleBinding | `rbac.authorization.k8s.io/v1` | RBAC |
| 20 | ClusterRole | `rbac.authorization.k8s.io/v1` | RBAC |
| 21 | ClusterRoleBinding | `rbac.authorization.k8s.io/v1` | RBAC |
| 22 | ResourceQuota | `v1` | Cluster admin |
| 23 | LimitRange | `v1` | Cluster admin |
| 24 | HorizontalPodAutoscaler | `autoscaling/v2` | Autoscaling |
| 25 | PodDisruptionBudget | `policy/v1` | Cluster admin |
| 26 | CustomResourceDefinition | `apiextensions.k8s.io/v1` | Extensibility |

Also included: **§20 the shared PodSpec / PodTemplateSpec schema** used by Pod, Deployment, ReplicaSet, StatefulSet, DaemonSet, and Job — since it accounts for most of the field surface area in workload manifests, it's documented once and referenced from each controller.

---

## 2. Universal Fields (every resource)

Every manifest, regardless of kind, shares this top-level envelope:

```yaml
apiVersion: <group/version>        # required — e.g. v1, apps/v1, batch/v1
kind: <Kind>                       # required — e.g. Deployment, Service
metadata:                          # required
  name: <string>                   # required (or generateName instead, for server-side name generation)
  generateName: <string>           # optional — prefix; server appends random suffix; mutually exclusive with name
  namespace: <string>              # optional (default: "default") — omit for cluster-scoped kinds (Namespace, ClusterRole, PV, etc.)
  labels:                          # optional — key/value map, used for selection/grouping
    <key>: <value>
  annotations:                     # optional — key/value map, non-identifying metadata (tooling, docs, config hints)
    <key>: <value>
  ownerReferences:                 # optional — usually generated; links object to a parent that garbage-collects it
    - apiVersion: <string>
      kind: <string>
      name: <string>
      uid: <string>
      controller: <bool>            # optional
      blockOwnerDeletion: <bool>    # optional
  finalizers:                      # optional — list of strings; blocks deletion until removed
    - <string>
  uid: <string>                    # generated
  resourceVersion: <string>        # generated — used for optimistic concurrency
  generation: <int>                # generated — bumped on spec changes
  creationTimestamp: <string>      # generated
  deletionTimestamp: <string>      # generated — set when delete is requested, before finalizers clear
  managedFields: [...]             # generated — server-side apply bookkeeping
spec: {...}                        # required for most kinds — desired state (schema differs per kind, see below)
status: {...}                      # generated — observed state, controller-populated, never set by hand
```

Standard recommended labels (not enforced, but conventional / used by tooling):
```yaml
metadata:
  labels:
    app.kubernetes.io/name: <string>
    app.kubernetes.io/instance: <string>
    app.kubernetes.io/version: <string>
    app.kubernetes.io/component: <string>
    app.kubernetes.io/part-of: <string>
    app.kubernetes.io/managed-by: <string>
```

---

## 3. Shared PodSpec / PodTemplateSpec Reference

Used verbatim (or nested under `spec.template.spec`) by Pod, Deployment, ReplicaSet, StatefulSet, DaemonSet, and Job.

```yaml
spec:
  # --- Scheduling & identity ---
  containers:                        # required — at least one
    - name: <string>                 # required, unique within pod
      image: <string>                # required (or inherited from resolved digest)
      imagePullPolicy: Always | IfNotPresent | Never   # optional (default: IfNotPresent, or Always if tag is :latest)
      command: [<string>, ...]       # optional — overrides image ENTRYPOINT
      args: [<string>, ...]          # optional — overrides image CMD
      workingDir: <string>           # optional
      ports:                         # optional
        - name: <string>             # optional, must be unique, used by Service targetPort
          containerPort: <int>       # required
          protocol: TCP | UDP | SCTP # optional (default: TCP)
          hostPort: <int>            # optional, rarely used
      env:                           # optional
        - name: <string>             # required
          value: <string>            # optional — literal value
          valueFrom:                 # optional — mutually exclusive with value
            fieldRef: {fieldPath: <string>}
            resourceFieldRef: {containerName: <string>, resource: <string>, divisor: <quantity>}
            configMapKeyRef: {name: <string>, key: <string>, optional: <bool>}
            secretKeyRef: {name: <string>, key: <string>, optional: <bool>}
      envFrom:                       # optional — bulk-import all keys
        - prefix: <string>           # optional
          configMapRef: {name: <string>, optional: <bool>}
          secretRef: {name: <string>, optional: <bool>}
      resources:                     # optional but strongly recommended
        requests: {cpu: <quantity>, memory: <quantity>, ephemeral-storage: <quantity>}
        limits:   {cpu: <quantity>, memory: <quantity>, ephemeral-storage: <quantity>}
        claims:                      # optional — Dynamic Resource Allocation (DRA), GA in 1.32+
          - name: <string>
      volumeMounts:                  # optional
        - name: <string>             # required, matches spec.volumes[].name
          mountPath: <string>        # required
          subPath: <string>          # optional
          subPathExpr: <string>      # optional — supports env var expansion
          readOnly: <bool>           # optional (default: false)
          mountPropagation: None | HostToContainer | Bidirectional  # optional
      volumeDevices:                 # optional — raw block devices
        - name: <string>
          devicePath: <string>
      livenessProbe: &probe          # optional
        httpGet: {path: <string>, port: <int|string>, host: <string>, scheme: HTTP|HTTPS, httpHeaders: [{name: <string>, value: <string>}]}
        tcpSocket: {port: <int|string>, host: <string>}
        exec: {command: [<string>, ...]}
        grpc: {port: <int>, service: <string>}         # optional
        initialDelaySeconds: <int>   # optional (default: 0)
        periodSeconds: <int>         # optional (default: 10)
        timeoutSeconds: <int>        # optional (default: 1)
        successThreshold: <int>      # optional (default: 1, must be 1 for liveness/startup)
        failureThreshold: <int>      # optional (default: 3)
        terminationGracePeriodSeconds: <int>  # optional
      readinessProbe: *probe         # optional — same schema as livenessProbe
      startupProbe: *probe           # optional — same schema, gates the other probes until it succeeds
      lifecycle:                     # optional
        postStart: {exec: {...} | httpGet: {...} | sleep: {seconds: <int>}}
        preStop:   {exec: {...} | httpGet: {...} | sleep: {seconds: <int>}}
      terminationMessagePath: <string>       # optional (default: /dev/termination-log)
      terminationMessagePolicy: File | FallbackToLogsOnError  # optional (default: File)
      securityContext:               # optional — container-level (see §3a)
        <see below>
      stdin: <bool>                  # optional
      stdinOnce: <bool>              # optional
      tty: <bool>                    # optional
      restartPolicy: Always          # optional — only meaningful for init containers (defines "sidecar" containers, 1.29+ stable)

  initContainers: [<same schema as containers>]   # optional — run sequentially to completion before containers start

  ephemeralContainers: [<same schema as containers, minus resources/ports mutability>]  # optional — for debugging live pods

  volumes:                           # optional
    - name: <string>                 # required
      # exactly one of the following source types:
      emptyDir: {medium: "" | Memory, sizeLimit: <quantity>}
      hostPath: {path: <string>, type: "" | DirectoryOrCreate | Directory | FileOrCreate | File | Socket | CharDevice | BlockDevice}
      configMap: {name: <string>, items: [{key: <string>, path: <string>, mode: <int>}], defaultMode: <int>, optional: <bool>}
      secret: {secretName: <string>, items: [...], defaultMode: <int>, optional: <bool>}
      persistentVolumeClaim: {claimName: <string>, readOnly: <bool>}
      projected: {sources: [{configMap: {...}, secret: {...}, downwardAPI: {...}, serviceAccountToken: {audience: <string>, expirationSeconds: <int>, path: <string>}}], defaultMode: <int>}
      downwardAPI: {items: [{path: <string>, fieldRef: {...}, resourceFieldRef: {...}}]}
      csi: {driver: <string>, readOnly: <bool>, fsType: <string>, volumeAttributes: {<key>: <value>}, nodePublishSecretRef: {name: <string>}}
      nfs: {server: <string>, path: <string>, readOnly: <bool>}
      iscsi: {...}
      fc: {...}
      # cloud-provider in-tree volume types (awsElasticBlockStore, gcePersistentDisk, azureDisk, etc.) are deprecated in favor of CSI drivers

  # --- Scheduling controls ---
  nodeName: <string>                 # optional — bypasses scheduler, pins directly
  nodeSelector: {<key>: <value>}     # optional — simple node label match
  affinity:                          # optional
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms: [{matchExpressions: [{key: <string>, operator: In|NotIn|Exists|DoesNotExist|Gt|Lt, values: [<string>]}], matchFields: [...]}]
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: <int 1-100>
          preference: {matchExpressions: [...], matchFields: [...]}
    podAffinity:                     # same shape as podAntiAffinity
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector: {matchLabels: {...}, matchExpressions: [...]}
          topologyKey: <string>
          namespaces: [<string>]
          namespaceSelector: {...}
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: <int 1-100>
          podAffinityTerm: {labelSelector: {...}, topologyKey: <string>}
    podAntiAffinity: {<same shape as podAffinity>}
  tolerations:                       # optional
    - key: <string>
      operator: Exists | Equal       # optional (default: Equal)
      value: <string>
      effect: NoSchedule | PreferNoSchedule | NoExecute
      tolerationSeconds: <int>       # optional — only for NoExecute
  topologySpreadConstraints:         # optional
    - maxSkew: <int>                 # required
      topologyKey: <string>          # required
      whenUnsatisfiable: DoNotSchedule | ScheduleAnyway  # required
      labelSelector: {...}
      minDomains: <int>              # optional
      nodeAffinityPolicy: Honor | Ignore  # optional
      nodeTaintsPolicy: Honor | Ignore    # optional
      matchLabelKeys: [<string>]     # optional
  schedulerName: <string>            # optional (default: default-scheduler)
  priorityClassName: <string>        # optional
  priority: <int>                    # generated from priorityClassName
  preemptionPolicy: PreemptLowerPriority | Never  # optional
  runtimeClassName: <string>         # optional — selects a RuntimeClass (e.g. gVisor, Kata)
  overhead: {cpu: <quantity>, memory: <quantity>}  # generated from RuntimeClass

  # --- Lifecycle & networking ---
  restartPolicy: Always | OnFailure | Never   # optional (default: Always; Job typically uses OnFailure/Never)
  terminationGracePeriodSeconds: <int>        # optional (default: 30)
  activeDeadlineSeconds: <int>                # optional
  dnsPolicy: ClusterFirst | ClusterFirstWithHostNet | Default | None  # optional (default: ClusterFirst)
  dnsConfig: {nameservers: [<string>], searches: [<string>], options: [{name: <string>, value: <string>}]}
  hostNetwork: <bool>                # optional (default: false)
  hostPID: <bool>                    # optional (default: false)
  hostIPC: <bool>                    # optional (default: false)
  hostname: <string>                 # optional
  subdomain: <string>                # optional
  hostAliases: [{ip: <string>, hostnames: [<string>]}]  # optional
  setHostnameAsFQDN: <bool>          # optional

  # --- Identity & access ---
  serviceAccountName: <string>       # optional (default: "default")
  automountServiceAccountToken: <bool>  # optional (default: true)
  imagePullSecrets: [{name: <string>}]  # optional

  # --- Security ---
  securityContext:                   # optional — pod-level (§3a)
    <see below>

  # --- Misc ---
  enableServiceLinks: <bool>         # optional (default: true)
  shareProcessNamespace: <bool>      # optional (default: false)
  readinessGates: [{conditionType: <string>}]  # optional
  os: {name: linux | windows}        # optional
  resourceClaims: [{name: <string>, source: {resourceClaimName: <string> | resourceClaimTemplateName: <string>}}]  # optional, DRA
```

### 3a. SecurityContext (Pod-level vs Container-level)

```yaml
# Pod-level (spec.securityContext) — sets defaults inherited by all containers
runAsUser: <int>
runAsGroup: <int>
runAsNonRoot: <bool>
fsGroup: <int>
fsGroupChangePolicy: Always | OnRootMismatch
supplementalGroups: [<int>]
seLinuxOptions: {level: <string>, role: <string>, type: <string>, user: <string>}
seccompProfile: {type: RuntimeDefault | Localhost | Unconfined, localhostProfile: <string>}
sysctls: [{name: <string>, value: <string>}]
windowsOptions: {gmsaCredentialSpecName: <string>, gmsaCredentialSpec: <string>, runAsUserName: <string>, hostProcess: <bool>}

# Container-level (containers[].securityContext) — overrides pod-level for that container
privileged: <bool>                   # optional (default: false)
allowPrivilegeEscalation: <bool>     # optional
readOnlyRootFilesystem: <bool>       # optional (default: false)
runAsUser: <int>
runAsGroup: <int>
runAsNonRoot: <bool>
capabilities: {add: [<string>], drop: [<string>]}
procMount: Default | Unmasked
seccompProfile: {...}
seLinuxOptions: {...}
```

### 3b. PodTemplateSpec wrapper (used inside controllers)

```yaml
spec:
  template:                          # required in all controllers below
    metadata:                        # labels here MUST match spec.selector
      labels: {<key>: <value>}
      annotations: {<key>: <value>}
    spec: {<full PodSpec from §3>}
```

---

## 4. Pod (`v1`)

Rarely created directly (controllers manage pods), but its `spec` is exactly §3.

```yaml
apiVersion: v1
kind: Pod
metadata: {<see §2>}
spec: {<full PodSpec, see §3>}
status:                              # generated
  phase: Pending | Running | Succeeded | Failed | Unknown
  conditions: [{type: PodScheduled|Initialized|ContainersReady|Ready, status: "True"|"False"|"Unknown", reason: <string>, message: <string>}]
  hostIP: <string>
  hostIPs: [{ip: <string>}]
  podIP: <string>
  podIPs: [{ip: <string>}]
  startTime: <string>
  qosClass: Guaranteed | Burstable | BestEffort
  containerStatuses: [{name: <string>, state: {waiting|running|terminated: {...}}, lastState: {...}, ready: <bool>, restartCount: <int>, image: <string>, imageID: <string>, started: <bool>}]
  initContainerStatuses: [...]
  ephemeralContainerStatuses: [...]
```

---

## 5. Deployment (`apps/v1`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {<see §2>}
spec:
  replicas: <int>                    # optional (default: 1)
  selector:                          # required — immutable after creation
    matchLabels: {<key>: <value>}
    matchExpressions: [{key: <string>, operator: In|NotIn|Exists|DoesNotExist, values: [<string>]}]
  template: {<PodTemplateSpec, see §3b>}  # required
  strategy:                          # optional
    type: RollingUpdate | Recreate   # optional (default: RollingUpdate)
    rollingUpdate:
      maxUnavailable: <int|percentage>  # optional (default: 25%)
      maxSurge: <int|percentage>        # optional (default: 25%)
  minReadySeconds: <int>             # optional (default: 0)
  revisionHistoryLimit: <int>        # optional (default: 10)
  paused: <bool>                     # optional (default: false)
  progressDeadlineSeconds: <int>     # optional (default: 600)
status:                              # generated
  observedGeneration: <int>
  replicas: <int>
  updatedReplicas: <int>
  readyReplicas: <int>
  availableReplicas: <int>
  unavailableReplicas: <int>
  conditions: [{type: Available|Progressing|ReplicaFailure, status: <string>, reason: <string>, message: <string>, lastUpdateTime: <string>, lastTransitionTime: <string>}]
  collisionCount: <int>
```

## 6. ReplicaSet (`apps/v1`)

Rarely authored directly — owned by Deployments. Same shape as Deployment minus `strategy`.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata: {<see §2>}
spec:
  replicas: <int>                    # optional (default: 1)
  selector: {<see Deployment>}       # required
  template: {<PodTemplateSpec>}      # required
  minReadySeconds: <int>             # optional
status:
  replicas: <int>
  fullyLabeledReplicas: <int>
  readyReplicas: <int>
  availableReplicas: <int>
  observedGeneration: <int>
  conditions: [...]
```

## 7. StatefulSet (`apps/v1`)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: {<see §2>}
spec:
  replicas: <int>                    # optional (default: 1)
  selector: {<see Deployment>}       # required
  template: {<PodTemplateSpec>}      # required
  serviceName: <string>              # required — headless Service governing network identity
  podManagementPolicy: OrderedReady | Parallel  # optional (default: OrderedReady)
  updateStrategy:
    type: RollingUpdate | OnDelete   # optional (default: RollingUpdate)
    rollingUpdate:
      partition: <int>               # optional (default: 0)
      maxUnavailable: <int|percentage>  # optional (beta)
  volumeClaimTemplates:               # optional — per-pod PVCs, stable identity
    - metadata: {name: <string>, labels: {...}, annotations: {...}}
      spec: {<PersistentVolumeClaimSpec, see §14>}
  minReadySeconds: <int>             # optional
  persistentVolumeClaimRetentionPolicy:  # optional (stable 1.27+)
    whenDeleted: Retain | Delete
    whenScaled: Retain | Delete
  revisionHistoryLimit: <int>        # optional (default: 10)
  ordinals: {start: <int>}           # optional — customize starting ordinal
status:
  observedGeneration: <int>
  replicas: <int>
  readyReplicas: <int>
  currentReplicas: <int>
  updatedReplicas: <int>
  currentRevision: <string>
  updateRevision: <string>
  collisionCount: <int>
  availableReplicas: <int>
  conditions: [...]
```

## 8. DaemonSet (`apps/v1`)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: {<see §2>}
spec:
  selector: {<see Deployment>}       # required
  template: {<PodTemplateSpec>}      # required
  updateStrategy:
    type: RollingUpdate | OnDelete   # optional (default: RollingUpdate)
    rollingUpdate:
      maxUnavailable: <int|percentage>  # optional (default: 1)
      maxSurge: <int|percentage>        # optional (default: 0)
  minReadySeconds: <int>             # optional
  revisionHistoryLimit: <int>        # optional (default: 10)
status:
  currentNumberScheduled: <int>
  numberMisscheduled: <int>
  desiredNumberScheduled: <int>
  numberReady: <int>
  observedGeneration: <int>
  updatedNumberScheduled: <int>
  numberAvailable: <int>
  numberUnavailable: <int>
  collisionCount: <int>
  conditions: [...]
```

## 9. Job (`batch/v1`)

```yaml
apiVersion: batch/v1
kind: Job
metadata: {<see §2>}
spec:
  template: {<PodTemplateSpec — restartPolicy must be OnFailure or Never>}  # required
  selector: {...}                    # generated — do not set manually (auto-generated from template labels)
  parallelism: <int>                 # optional (default: 1)
  completions: <int>                 # optional (default: 1; omit for "work queue" pattern)
  completionMode: NonIndexed | Indexed  # optional (default: NonIndexed)
  backoffLimit: <int>                # optional (default: 6)
  backoffLimitPerIndex: <int>        # optional — Indexed jobs only
  maxFailedIndexes: <int>            # optional — Indexed jobs only
  activeDeadlineSeconds: <int>       # optional
  ttlSecondsAfterFinished: <int>     # optional — auto-cleanup after completion
  suspend: <bool>                    # optional (default: false)
  podFailurePolicy:                  # optional
    rules:
      - action: FailJob | Ignore | Count | FailIndex
        onExitCodes: {containerName: <string>, operator: In|NotIn, values: [<int>]}
        onPodConditions: [{type: <string>, status: <string>}]
  podReplacementPolicy: TerminatingOrFailed | Failed  # optional
  manualSelector: <bool>             # optional (default: false)
status:
  active: <int>
  succeeded: <int>
  failed: <int>
  startTime: <string>
  completionTime: <string>
  conditions: [{type: Suspended|Complete|Failed|FailureTarget, status: <string>, reason: <string>, message: <string>}]
  ready: <int>
  terminating: <int>
  completedIndexes: <string>
  uncountedTerminatedPods: {...}
```

## 10. CronJob (`batch/v1`)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: {<see §2>}
spec:
  schedule: <string>                 # required — standard cron syntax, e.g. "*/5 * * * *"
  timeZone: <string>                 # optional — IANA tz name, e.g. "America/New_York" (stable 1.27+)
  jobTemplate:                       # required
    metadata: {labels: {...}, annotations: {...}}
    spec: {<full JobSpec, see §9>}
  concurrencyPolicy: Allow | Forbid | Replace  # optional (default: Allow)
  startingDeadlineSeconds: <int>     # optional
  suspend: <bool>                    # optional (default: false)
  successfulJobsHistoryLimit: <int>  # optional (default: 3)
  failedJobsHistoryLimit: <int>      # optional (default: 1)
status:
  active: [{name: <string>, namespace: <string>, uid: <string>}]
  lastScheduleTime: <string>
  lastSuccessfulTime: <string>
```

---

## 11. Service (`v1`)

```yaml
apiVersion: v1
kind: Service
metadata: {<see §2>}
spec:
  type: ClusterIP | NodePort | LoadBalancer | ExternalName  # optional (default: ClusterIP)
  selector: {<key>: <value>}         # optional — omit for Services without selectors (manual Endpoints, or ExternalName)
  ports:
    - name: <string>                 # optional, required if >1 port
      protocol: TCP | UDP | SCTP     # optional (default: TCP)
      appProtocol: <string>          # optional — e.g. http, https, grpc
      port: <int>                    # required — Service's own port
      targetPort: <int|string>       # optional (default: same as port) — container port or named port
      nodePort: <int>                # optional — only for type NodePort/LoadBalancer; auto-allocated if unset
  clusterIP: <string> | None         # optional — "None" makes it headless
  clusterIPs: [<string>]             # optional — dual-stack
  ipFamilies: [IPv4 | IPv6]          # optional
  ipFamilyPolicy: SingleStack | PreferDualStack | RequireDualStack  # optional
  externalIPs: [<string>]            # optional
  externalName: <string>             # required if type: ExternalName
  externalTrafficPolicy: Cluster | Local  # optional
  internalTrafficPolicy: Cluster | Local  # optional (default: Cluster)
  sessionAffinity: None | ClientIP   # optional (default: None)
  sessionAffinityConfig: {clientIP: {timeoutSeconds: <int>}}  # optional
  loadBalancerClass: <string>        # optional
  loadBalancerIP: <string>           # optional, deprecated — use provider-specific annotations
  loadBalancerSourceRanges: [<string>]  # optional
  allocateLoadBalancerNodePorts: <bool>  # optional (default: true)
  publishNotReadyAddresses: <bool>   # optional (default: false)
  healthCheckNodePort: <int>         # optional — auto-set when externalTrafficPolicy: Local
status:
  loadBalancer:
    ingress: [{ip: <string>, hostname: <string>, ports: [{port: <int>, protocol: <string>, error: <string>}]}]
  conditions: [...]
```

## 12. Ingress (`networking.k8s.io/v1`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:                       # optional — ingress-controller-specific behavior (nginx, ALB, etc.)
    <controller-specific-key>: <value>
spec:
  ingressClassName: <string>         # optional — selects the IngressClass/controller
  defaultBackend:                    # optional — fallback for unmatched requests
    service: {name: <string>, port: {number: <int> | name: <string>}}
    resource: {apiGroup: <string>, kind: <string>, name: <string>}
  tls:                                # optional
    - hosts: [<string>]
      secretName: <string>           # references a kubernetes.io/tls Secret
  rules:                              # optional
    - host: <string>                 # optional — omit to match any host
      http:
        paths:
          - path: <string>           # required
            pathType: Exact | Prefix | ImplementationSpecific  # required
            backend:
              service: {name: <string>, port: {number: <int> | name: <string>}}
              resource: {apiGroup: <string>, kind: <string>, name: <string>}
status:
  loadBalancer:
    ingress: [{ip: <string>, hostname: <string>}]
```

## 13. NetworkPolicy (`networking.k8s.io/v1`)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {<see §2>}
spec:
  podSelector: {matchLabels: {...}, matchExpressions: [...]}  # required — {} selects all pods in namespace
  policyTypes: [Ingress, Egress]     # optional — inferred from presence of ingress/egress if unset
  ingress:                            # optional — omit entirely to deny all ingress; [] with policyTypes:[Ingress] also denies all
    - from:
        - podSelector: {...}
          namespaceSelector: {...}
          ipBlock: {cidr: <string>, except: [<string>]}
      ports:
        - protocol: TCP | UDP | SCTP
          port: <int|string>
          endPort: <int>              # optional — port range
  egress:                             # optional — same shape as ingress but "to" instead of "from"
    - to:
        - podSelector: {...}
          namespaceSelector: {...}
          ipBlock: {...}
      ports: [...]
```

---

## 14. ConfigMap (`v1`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata: {<see §2>}
data:                                 # optional — UTF-8 string values
  <key>: <string>
  # multi-line example:
  # app.properties: |
  #   key1=value1
  #   key2=value2
binaryData:                           # optional — base64-encoded binary values
  <key>: <base64-string>
immutable: <bool>                     # optional (default: false) — prevents updates, improves apiserver perf
```

## 15. Secret (`v1`)

```yaml
apiVersion: v1
kind: Secret
metadata: {<see §2>}
type: Opaque | kubernetes.io/service-account-token | kubernetes.io/dockercfg | kubernetes.io/dockerconfigjson | kubernetes.io/basic-auth | kubernetes.io/ssh-auth | kubernetes.io/tls | bootstrap.kubernetes.io/token | <custom>
  # optional (default: Opaque)
data:                                  # optional — base64-encoded values (required encoding, NOT encryption)
  <key>: <base64-string>
stringData:                            # optional — write-only plaintext convenience field; merged into data on write
  <key>: <string>
immutable: <bool>                      # optional (default: false)
```

Common `type: kubernetes.io/tls` shape:
```yaml
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>
```
Common `type: kubernetes.io/dockerconfigjson` shape:
```yaml
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-json>
```

---

## 16. PersistentVolume (`v1`) — cluster-scoped

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: {<see §2, no namespace>}
spec:
  capacity: {storage: <quantity>}    # required
  volumeMode: Filesystem | Block     # optional (default: Filesystem)
  accessModes: [ReadWriteOnce | ReadOnlyMany | ReadWriteMany | ReadWriteOncePod]  # required
  persistentVolumeReclaimPolicy: Retain | Delete | Recycle  # optional (default: Retain, dynamic default: Delete)
  storageClassName: <string>         # optional
  mountOptions: [<string>]           # optional
  nodeAffinity:                      # optional — for local/topology-restricted volumes
    required: {nodeSelectorTerms: [...]}
  # exactly one volume source, most commonly:
  csi: {driver: <string>, volumeHandle: <string>, readOnly: <bool>, fsType: <string>, volumeAttributes: {...}, controllerPublishSecretRef: {...}, nodeStageSecretRef: {...}, nodePublishSecretRef: {...}}
  hostPath: {path: <string>, type: <string>}
  nfs: {server: <string>, path: <string>, readOnly: <bool>}
  local: {path: <string>, fsType: <string>}
  claimRef: {name: <string>, namespace: <string>, uid: <string>}  # generated on bind
status:
  phase: Pending | Available | Bound | Released | Failed
  message: <string>
  reason: <string>
  lastPhaseTransitionTime: <string>
```

## 17. PersistentVolumeClaim (`v1`)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {<see §2>}
spec:
  accessModes: [ReadWriteOnce | ReadOnlyMany | ReadWriteMany | ReadWriteOncePod]  # required
  volumeMode: Filesystem | Block     # optional (default: Filesystem)
  resources:
    requests: {storage: <quantity>}  # required
    limits: {storage: <quantity>}    # optional
  selector: {matchLabels: {...}, matchExpressions: [...]}  # optional
  storageClassName: <string>         # optional — omit to use cluster default; "" to disable dynamic provisioning
  volumeName: <string>               # optional — bind to a specific pre-created PV
  dataSource: {name: <string>, kind: PersistentVolumeClaim | VolumeSnapshot, apiGroup: <string>}  # optional — clone/snapshot
  dataSourceRef: {name: <string>, kind: <string>, apiGroup: <string>, namespace: <string>}  # optional
  volumeAttributesClassName: <string>  # optional (beta) — mutable volume attributes
status:
  phase: Pending | Bound | Lost
  accessModes: [...]
  capacity: {storage: <quantity>}
  conditions: [...]
  allocatedResources: {storage: <quantity>}
  currentVolumeAttributesClassName: <string>
```

## 18. StorageClass (`storage.k8s.io/v1`) — cluster-scoped

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {<see §2, no namespace>}
provisioner: <string>                # required — e.g. kubernetes.io/aws-ebs, ebs.csi.aws.com, pd.csi.storage.gke.io
parameters: {<key>: <value>}         # optional — provisioner-specific
reclaimPolicy: Retain | Delete       # optional (default: Delete)
allowVolumeExpansion: <bool>         # optional (default: false)
volumeBindingMode: Immediate | WaitForFirstConsumer  # optional (default: Immediate)
mountOptions: [<string>]             # optional
allowedTopologies:                   # optional
  - matchLabelExpressions: [{key: <string>, values: [<string>]}]
```

---

## 19. Namespace (`v1`) — cluster-scoped

```yaml
apiVersion: v1
kind: Namespace
metadata: {<see §2, no namespace field>}
spec:
  finalizers: [kubernetes]           # optional
status:
  phase: Active | Terminating
  conditions: [...]
```

## 20. ServiceAccount (`v1`)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: {<see §2>}
secrets: [{name: <string>}]          # optional, legacy — token Secrets are auto-created via TokenRequest now
imagePullSecrets: [{name: <string>}] # optional — auto-attached to pods using this SA
automountServiceAccountToken: <bool> # optional (default: true)
```

## 21. Role (`rbac.authorization.k8s.io/v1`) — namespaced

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {<see §2>}
rules:
  - apiGroups: [<string>]            # required — "" for core group
    resources: [<string>]            # required — e.g. pods, deployments, "pods/log"
    resourceNames: [<string>]        # optional — restrict to specific named objects
    verbs: [get, list, watch, create, update, patch, delete, deletecollection]  # required
    nonResourceURLs: [<string>]      # optional — ClusterRole only, e.g. "/healthz"
```

## 22. RoleBinding (`rbac.authorization.k8s.io/v1`) — namespaced

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {<see §2>}
subjects:                             # required
  - kind: User | Group | ServiceAccount
    name: <string>
    namespace: <string>               # required if kind: ServiceAccount
    apiGroup: rbac.authorization.k8s.io  # required for User/Group; omit for ServiceAccount
roleRef:                              # required, immutable
  apiGroup: rbac.authorization.k8s.io
  kind: Role | ClusterRole
  name: <string>
```

## 23. ClusterRole (`rbac.authorization.k8s.io/v1`) — cluster-scoped

Same `rules` schema as Role, plus optional `aggregationRule`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: {<see §2, no namespace>}
rules: [<same as Role>]
aggregationRule:                      # optional — auto-compose rules from other ClusterRoles
  clusterRoleSelectors: [{matchLabels: {...}}]
```

## 24. ClusterRoleBinding (`rbac.authorization.k8s.io/v1`) — cluster-scoped

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: {<see §2, no namespace>}
subjects: [<same as RoleBinding>]
roleRef: {apiGroup: rbac.authorization.k8s.io, kind: ClusterRole, name: <string>}
```

---

## 25. ResourceQuota (`v1`)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: {<see §2>}
spec:
  hard:                               # required
    requests.cpu: <quantity>
    requests.memory: <quantity>
    limits.cpu: <quantity>
    limits.memory: <quantity>
    requests.storage: <quantity>
    persistentvolumeclaims: <int>
    pods: <int>
    services: <int>
    services.loadbalancers: <int>
    services.nodeports: <int>
    count/<resource>.<group>: <int>   # e.g. count/deployments.apps
    <storageclass-name>.storageclass.storage.k8s.io/requests.storage: <quantity>
  scopes: [Terminating | NotTerminating | BestEffort | NotBestEffort | PriorityClass | CrossNamespacePodAffinity]  # optional
  scopeSelector:                      # optional
    matchExpressions: [{scopeName: <string>, operator: In|NotIn|Exists|DoesNotExist, values: [<string>]}]
status:
  hard: {...}                         # generated
  used: {...}                         # generated
```

## 26. LimitRange (`v1`)

```yaml
apiVersion: v1
kind: LimitRange
metadata: {<see §2>}
spec:
  limits:
    - type: Container | Pod | PersistentVolumeClaim  # required
      default: {cpu: <quantity>, memory: <quantity>}       # optional — default limits
      defaultRequest: {cpu: <quantity>, memory: <quantity>}  # optional — default requests
      min: {cpu: <quantity>, memory: <quantity>}            # optional
      max: {cpu: <quantity>, memory: <quantity>}            # optional
      maxLimitRequestRatio: {cpu: <quantity>, memory: <quantity>}  # optional
```

---

## 27. HorizontalPodAutoscaler (`autoscaling/v2`)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {<see §2>}
spec:
  scaleTargetRef: {apiVersion: <string>, kind: Deployment|StatefulSet|ReplicaSet, name: <string>}  # required
  minReplicas: <int>                  # optional (default: 1)
  maxReplicas: <int>                  # required
  metrics:                            # optional — defaults to 80% CPU if omitted
    - type: Resource | Pods | Object | External | ContainerResource
      resource: {name: cpu|memory, target: {type: Utilization|AverageValue, averageUtilization: <int>, averageValue: <quantity>}}
      pods: {metric: {name: <string>, selector: {...}}, target: {type: AverageValue, averageValue: <quantity>}}
      object: {describedObject: {apiVersion: <string>, kind: <string>, name: <string>}, metric: {name: <string>}, target: {type: Value|AverageValue, value: <quantity>}}
      external: {metric: {name: <string>, selector: {...}}, target: {type: Value|AverageValue, value: <quantity>}}
      containerResource: {name: cpu|memory, container: <string>, target: {...}}
  behavior:                           # optional — fine-tune scale up/down speed
    scaleUp: &scaleBehavior
      stabilizationWindowSeconds: <int>
      selectPolicy: Max | Min | Disabled
      policies: [{type: Pods|Percent, value: <int>, periodSeconds: <int>}]
    scaleDown: *scaleBehavior
status:
  observedGeneration: <int>
  lastScaleTime: <string>
  currentReplicas: <int>
  desiredReplicas: <int>
  currentMetrics: [...]
  conditions: [...]
```

## 28. PodDisruptionBudget (`policy/v1`)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {<see §2>}
spec:
  selector: {matchLabels: {...}, matchExpressions: [...]}  # required
  minAvailable: <int|percentage>      # optional — mutually exclusive with maxUnavailable
  maxUnavailable: <int|percentage>    # optional
  unhealthyPodEvictionPolicy: IfHealthyBudget | AlwaysAllow  # optional (default: IfHealthyBudget)
status:
  observedGeneration: <int>
  disruptionsAllowed: <int>
  currentHealthy: <int>
  desiredHealthy: <int>
  expectedPods: <int>
  conditions: [...]
```

---

## 29. CustomResourceDefinition (`apiextensions.k8s.io/v1`) — cluster-scoped

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: <plural>.<group>              # required — must match spec.names.plural + spec.group
spec:
  group: <string>                     # required — e.g. "example.com"
  scope: Namespaced | Cluster         # required
  names:                               # required
    plural: <string>                  # required
    singular: <string>                # optional
    kind: <string>                    # required
    listKind: <string>                # optional
    shortNames: [<string>]            # optional
    categories: [<string>]            # optional
  versions:                            # required — at least one
    - name: <string>                  # required — e.g. v1, v1alpha1, v1beta1
      served: <bool>                  # required
      storage: <bool>                 # required — exactly one version must be true
      deprecated: <bool>              # optional
      deprecationWarning: <string>    # optional
      schema:
        openAPIV3Schema: {type: object, properties: {...}, required: [...]}  # required — full OpenAPI v3 schema
      subresources:                    # optional
        status: {}                     # enables /status subresource
        scale: {specReplicasPath: <string>, statusReplicasPath: <string>, labelSelectorPath: <string>}
      additionalPrinterColumns:        # optional — kubectl get custom columns
        - name: <string>
          type: string|integer|number|boolean|date
          jsonPath: <string>
          description: <string>
          priority: <int>
  conversion:                          # optional
    strategy: None | Webhook
    webhook: {clientConfig: {...}, conversionReviewVersions: [<string>]}
  preserveUnknownFields: <bool>        # optional, deprecated (default: false)
status:                                # generated
  conditions: [{type: Established|NamesAccepted|Terminating, status: <string>}]
  acceptedNames: {plural: <string>, singular: <string>, kind: <string>, ...}
  storedVersions: [<string>]
```

An actual **custom resource instance** created from such a CRD then looks like:
```yaml
apiVersion: <group>/<version>          # e.g. example.com/v1
kind: <Kind>                            # matches spec.names.kind above
metadata: {<see §2>}
spec: {<whatever your openAPIV3Schema defines>}
status: {<if subresources.status enabled, controller-managed>}
```

---

## 30. Cross-Cutting Syntax Notes

**Resource quantities** (`<quantity>`): decimal or binary SI suffixes — `128Mi`, `1Gi`, `500m` (millicpu), `2` (2 whole CPUs), `100M`, `1000Ki`. Binary (`Ki/Mi/Gi/Ti`) = powers of 1024; decimal (`k/M/G/T`) = powers of 1000.

**Label/selector rules:** keys are `[prefix/]name`, prefix ≤253 chars (DNS subdomain), name ≤63 chars, alphanumeric plus `-_.`, must start/end alphanumeric. Values follow the same 63-char rule, may be empty.

**Percentage-or-int fields** (e.g. `maxUnavailable`, `minAvailable`): accept either a bare integer (`2`) or a string percentage (`"25%"`).

**API deprecation notes** (things to avoid in new manifests):
- `apps/v1beta1`, `apps/v1beta2`, `extensions/v1beta1` — all removed; use `apps/v1`.
- `networking.k8s.io/v1beta1` Ingress — removed; use `networking.k8s.io/v1` (`pathType` is required).
- `batch/v1beta1` CronJob — removed; use `batch/v1`.
- `policy/v1beta1` PodDisruptionBudget/PodSecurityPolicy — PDB moved to `policy/v1`; PodSecurityPolicy was removed entirely (replaced by Pod Security Admission / `pod-security.kubernetes.io/*` namespace labels).
- In-tree cloud volume plugins (`awsElasticBlockStore`, `gcePersistentDisk`, `azureDisk/File`, `cinder`, etc.) are deprecated in favor of CSI equivalents.

---

## 31. Master Field-Checklist (quick authoring pass)

Use this when writing a new manifest from scratch — go top to bottom, skip what doesn't apply:

- [ ] `apiVersion` + `kind` correct for target resource and cluster version
- [ ] `metadata.name` (or `generateName`) set, DNS-1123 compliant
- [ ] `metadata.namespace` set (namespaced kinds only)
- [ ] `metadata.labels` set — include `app.kubernetes.io/*` conventions
- [ ] Workloads: `spec.selector` matches `spec.template.metadata.labels` exactly
- [ ] Containers: `image` pinned to a tag or digest (avoid bare `:latest` in production)
- [ ] Containers: `resources.requests` and `resources.limits` set
- [ ] Containers: `livenessProbe` / `readinessProbe` (and `startupProbe` for slow-starting apps)
- [ ] Containers: `securityContext` — `runAsNonRoot`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]` where feasible
- [ ] Secrets: not embedded as plaintext env values in ConfigMaps; use `Secret` + `secretKeyRef`
- [ ] Services: `selector` matches pod labels; `targetPort` matches container port name/number
- [ ] Ingress: `pathType` set on every path; `ingressClassName` set explicitly (no implicit default)
- [ ] PVCs: `accessModes` and `storageClassName` match what the backing storage actually supports
- [ ] RBAC: least-privilege `verbs`/`resources` — avoid `resources: ["*"]` / `verbs: ["*"]` unless truly needed
- [ ] HPA: `minReplicas`/`maxReplicas` sane relative to `resources.requests`
- [ ] `terminationGracePeriodSeconds` reviewed for stateful workloads
- [ ] `topologySpreadConstraints` or `podAntiAffinity` set for HA workloads across zones/nodes
- [ ] `revisionHistoryLimit` / `ttlSecondsAfterFinished` set to bound object growth
- [ ] Namespace-scoped `ResourceQuota` / `LimitRange` accounted for if the cluster enforces them

---

*Reference compiled for Kubernetes 1.30–1.33 stable APIs. Always cross-check exact field availability against `kubectl explain <kind>.<field> --recursive` for your specific cluster version, since alpha/beta fields and gate-flagged features vary by version and cluster configuration.*
