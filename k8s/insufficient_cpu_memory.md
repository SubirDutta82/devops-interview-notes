
```
Pods stay stuck in `Pending` with "Insufficient cpu" / "Insufficient memory" events for a handful of distinct underlying reasons. Here's a breakdown of the common causes:

## 1. Cluster genuinely lacks capacity
- No node has enough allocatable CPU/memory for the pod's `requests` — not the actual usage, but the sum of all requests already scheduled on each node.
- Cluster autoscaler is absent, disabled, or capped (`maxNodes` already reached), so no new node gets added to relieve pressure.
- Node pool/instance type mismatch — the pod needs a GPU, large memory, or specific instance family that doesn't exist in any node group.

## 2. Requests are set too high (over-provisioning)
- Developers set generous `resources.requests` "just to be safe," and no single node can satisfy that request even though real usage is low.
- Vertical Pod Autoscaler or a manifest template inflated requests without anyone noticing.

## 3. Resource fragmentation
- Enough total free CPU/memory exists across the cluster, but it's scattered in small pockets on many nodes — no single node has enough contiguous allocatable capacity for one pod.

## 4. Node-level reservations eating into allocatable resources
- `kube-reserved`, `system-reserved`, and eviction thresholds reduce "allocatable" below what raw node capacity suggests.
- DaemonSets (logging agents, CNI, service mesh sidecar injectors) consume requests on every node, shrinking what's left for regular pods.

## 5. Taints, tolerations, affinity/anti-affinity, and topology constraints
- Nodes with free capacity exist but are tainted (e.g., dedicated node pools) and the pod lacks matching tolerations.
- nodeSelector / nodeAffinity restricts scheduling to a subset of nodes that happen to be full.
- Pod anti-affinity or topology spread constraints prevent packing pods together even when raw resources are available.
- PodDisruptionBudgets combined with autoscaler scale-down can create races where capacity briefly disappears.

## 6. Namespace or cluster-level policy limits
- ResourceQuota on the namespace caps total CPU/memory requests — even if nodes have room, the pod is blocked at admission/scheduling.
- LimitRange defaults conflict with what the pod needs.

## 7. Autoscaler is present but too slow or broken
- Scale-up is in progress but takes minutes (cloud provider provisioning delay) — pod looks "stuck" but will resolve.
- Autoscaler is misconfigured: wrong IAM permissions, cloud API quota exhausted (e.g., vCPU quota limit on AWS/GCP/Azure), so it tries to add nodes and fails silently.
- Autoscaler's node group templates don't match what the scheduler thinks it needs (label/taint mismatch between ASG and actual node config).

## 8. Bin-packing / scheduler scoring effects
- Scheduler filters out nodes correctly but scoring plugins (e.g., `NodeResourcesFit`, `PodTopologySpread`) combined with many competing pending pods create a persistent queue where this particular pod never wins the scheduling race.

## 9. Pending pods hidden by unrelated PVC/binding issues
- Sometimes reported as "Insufficient CPU/Memory" but the actual root blocker is an unbound PersistentVolumeClaim or an image pull happening after scheduling — check the full event list, not just the top line, since multiple predicates can fail simultaneously and only the resource one gets displayed.

---

### How to diagnose quickly

kubectl describe pod <pod-name>       # look at Events section, "FailedScheduling"
kubectl describe nodes | grep -A5 "Allocated resources"
kubectl get resourcequota -n <namespace>
kubectl get pdb -n <namespace>

```
