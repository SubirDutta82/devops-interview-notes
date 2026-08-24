# The Ultimate Enterprise Kubernetes Sandbox: Scenario-Driven Failure Lab

This comprehensive blueprint outlines how to build a production-grade, highly resilient local simulation stack using KinD (Kubernetes in Docker), WSL2 (Ubuntu), and open-source cloud-native engineering tools. It allows you to simulate, track, and remediate nearly any production failure condition on demand for high-retention video content creation.

---

## 🏗️ 1. Complete Lab Architecture & Topology

```
                  ┌──────────────────────────────────────────┐
                  │ Windows Host OS (OBS Studio / Browser)   │
                  └────────────────────┬─────────────────────┘
                                       │ (Local Port Forwards)
                                       ▼
                  ┌──────────────────────────────────────────┐
                  │    WSL2 (Ubuntu Engine Environment)      │
                  └────────────────────┬─────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│              KinD Multi-Node Engine Template (Docker Network)             │
│                                                                          │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────┐  │
│  │ control-plane Node   │ │ worker-1 Node        │ │ worker-2 Node    │  │
│  │                      │ │                      │ │                  │  │
│  │ ─ NGINX Ingress      │ │ ─ App Workloads      │ │ ─ App Workloads  │  │
│  │ ─ Kube-API Server    │ │ ─ Prometheus Agents  │ │ ─ Storage Claims │  │
│  │ ─ Local Control Loops│ │ ─ Chaos Agents       │ │ ─ Chaos Agents   │  │
│  └──────────────────────┘ └──────────────────────┘ └──────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ Shared Cluster Infrastructure Namespaces                           │  │
│  │                                                                    │  │
│  │  [monitoring]   ▶ Prometheus Server, Alertmanager, Grafana Engine │  │
│  │  [chaos-mesh]   ▶ Ingress CRD Controllers, Kernel DaemonSets       │  │
│  │  [metallb-system] ▶ Layer 2 Load Balancer IP Pools (Dynamic Routing)│  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ 2. The Comprehensive Manifest Stack

### Core Cluster Provisioner File (`kind-infra-lab.yaml`)
Save this file inside your Ubuntu directory. It provisions an authentic distributed environment mapping local host HTTP/HTTPS ingress channels through the primary control plane.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: enterprise-failure-lab
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
```

**Execution Command:**
```bash
kind create cluster --config kind-infra-lab.yaml
```

### Advanced Networking Add-on: MetalLB Ingestion
To eliminate the `Pending` state for local load balancers, apply the standard manifest controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

Wait 60 seconds for pods to enter a `Running` state, then check your docker subnet address scope using `docker network inspect kind` to find your bridge range (usually `172.18.0.0/16`). Deploy the internal IP allocator (`metallb-config.yaml`):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: local-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: local-advertisement
  namespace: metallb-system
```
```bash
kubectl apply -f metallb-config.yaml
```

### Standard Enterprise Storage Class Configuration
Verify the dynamic storage provisioner layout:
```bash
kubectl get storageclass standard -o yaml
```

---

## 📈 3. Observability & Chaos Injection Engines

### Observability Deployment
Deploy the standard Prometheus stack using Helm to establish deep telemetry tracking:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install telemetry prometheus-community/kube-prometheus-stack   --namespace monitoring   --create-namespace   --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false   --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

### Chaos Engineering Controller Deployment
Install Chaos Mesh directly into your sandbox cluster to unlock instant, visual failure triggers via a browser UI:

```bash
curl -sSL https://mirrors.chaos-mesh.org/v2.6.2/install.sh | bash
```

To access the web interface safely inside Chrome or Edge on Windows, port-forward the front-end dashboard service:
```bash
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 8080:2333
```
Open your browser on Windows and navigate to: `http://localhost:8080`

---

## 🔬 4. Master Matrix: Simulating Any Scenario On Demand

| Error / Failure State | Root Cause Vector | Quick Production Injection Script / Manifest |
| :--- | :--- | :--- |
| **CrashLoopBackOff** | Misconfigured entrypoint, missing configs, or runtime app panic. | ```yaml
apiVersion: v1
kind: Pod
metadata:
  name: target-crashloop
spec:
  containers:
  - name: broken-app
    image: alpine
    command: ["/bin/sh", "-c", "sleep 5 && exit 1"]
``` |
| **OOMKilled (Exit 137)** | Memory utilization exceeding the maximum container limit. | ```yaml
apiVersion: v1
kind: Pod
metadata:
  name: target-oom
spec:
  containers:
  - name: ram-hog
    image: polinux/stress
    command: ["stress", "--vm", "1", "--vm-bytes", "50M", "--timeout", "30s"]
    resources:
      limits:
        memory: "20Mi"
``` |
| **CreateContainerConfigError** | Missing referenced ConfigMaps or Secrets during initialization. | ```yaml
apiVersion: v1
kind: Pod
metadata:
  name: target-config-error
spec:
  containers:
  - name: web-app
    image: nginx
    envFrom:
    - configMapRef:
        name: non-existent-configmap
``` |
| **ImagePullBackOff** | Typo in image name, missing tag, or unauthenticated private registry. | ```yaml
apiVersion: v1
kind: Pod
metadata:
  name: target-pull-error
spec:
  containers:
  - name: core-app
    image: nginx:this-tag-does-not-exist
``` |
| **FailedCreatePodSandBox** | CNI subnet conflicts or total IP exhaustion on a specific node. | **Via Chaos Mesh Dashboard:** Go to *New Experiment* -> *Network Attack* -> *Partition*. Isolate worker node network interfaces completely. |
| **DiskPressure / Evicted** | Host machine disk storage crossing the 85-90% alert threshold. | ```yaml
apiVersion: v1
kind: Pod
metadata:
  name: target-disk-pressure
spec:
  containers:
  - name: disk-hog
    image: alpine
    command: ["/bin/sh", "-c", "dd if=/dev/zero of=/data/largefile bs=1M count=5000 && sleep 300"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: enterprise-pvc
``` |
| **Network Latency / 504 Gateway** | Inter-service routing delays caused by flaky network lines or connection overloads. | **Via Chaos Mesh Custom Resource (CRD):**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - default
  delay:
    latency: '3000ms'
    jitter: '100ms'
  duration: '60s'
``` |

---

## 🎥 5. Creator Pro-Tips: Setting Up High-Retention Video Screens

### The Ultimate Dual-Pane Streaming Viewport
Launch `tmux` inside your Ubuntu console window and create a split pane (`Ctrl + b` followed by `"`). Set your screen views exactly like this before you start recording in OBS Studio:

```
┌──────────────────────────────────────────────────────────────────────────┐
│ TMUX OPERATIONAL CONSOLE (VISIBLE ON YOUTUBE VIDEO SCREEN)               │
├──────────────────────────────────────────────────────────────────────────┤
│ 🖥️ TOP PANEL: Visual System Logs & Pod Tally Status                     │
│ Command: k9s --namespace default                                         │
│ (Allows viewers to watch your pod turn bright amber or flashing red)    │
├──────────────────────────────────────────────────────────────────────────┤
│ 💻 BOTTOM PANEL: Active Failure Trigger / Troubleshooting Command Space  │
│ Command: kubectl apply -f failure-manifest.yaml                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### The Ultimate Video Sequence Script Framework
1. **The Hook (First 15 seconds):** "Our database server just dropped off the grid, Grafana alerts are flashing critical red, and our users are seeing 504 Gateways. Let's fix it."
2. **The Discovery (Next 90 seconds):** Split your screen. Run `kubectl get events --sort-by='.metadata.creationTimestamp'` or launch `k9s`. Show your viewers exactly *where* to trace the failure footprint.
3. **The Root Cause Isolate:** Move to your visual whiteboard (like Excalidraw) for 45 seconds. Map out the broken path (e.g., how an aggressive liveness probe caused an infinite restart loop).
4. **The Remediate Phase:** Apply your working YAML solution patch. Cut directly back to your Grafana dashboard view to show the traffic line curving back down to normal health metrics.

---

This framework is built entirely for informational and instructional reference. AI environments and package dependencies can evolve; consult official cloud-native tool documentation to maintain individual image versions.
