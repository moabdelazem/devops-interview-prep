# Kubernetes -- Scenario-Based Questions

Real-world troubleshooting and problem-solving exercises for junior DevOps/SysAdmin interviews. Each scenario lists all possible causes you should mention to the interviewer.

---

## Pod Issues

### Scenario 1: Pod stuck in CrashLoopBackOff

**Situation:** A Deployment's Pods are in `CrashLoopBackOff`. The application worked fine locally.

**How would you troubleshoot this?**

**Answer:**

**Step 1 -- Check Pod events:**
```bash
kubectl describe pod <pod-name>
```

**Step 2 -- Check container logs (current and previous):**
```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| OOMKilled (out of memory) | `Last State: OOMKilled` in describe output | Increase `resources.limits.memory` |
| Missing env var or ConfigMap/Secret | Error in logs referencing undefined variable; event shows `CreateContainerConfigError` | Create the missing ConfigMap/Secret or fix the reference |
| Wrong command or entrypoint | Logs show `exec format error` or `not found` | Fix `command`/`args` in the Pod spec or rebuild the image |
| Application crash on startup | Stack trace or error in logs | Fix the application bug; check config files and dependencies |
| Incorrect image tag | `ImagePullBackOff` followed by crash | Fix the image tag; verify the image runs locally |
| Read-only filesystem | `Read-only file system` in logs | Add a writable volume or set `readOnlyRootFilesystem: false` |
| Missing dependency (DB, API) | Connection refused or timeout errors in logs | Ensure dependency is running; use init containers to wait |
| Liveness probe failing too early | Pod starts then gets killed repeatedly | Add a `startupProbe` or increase `initialDelaySeconds` on the liveness probe |
| File permission issues | `Permission denied` in logs | Fix volume permissions or run as the correct user |

---

### Scenario 2: Pod stuck in Pending

**Situation:** A Pod stays in `Pending` state and never gets scheduled.

**How would you troubleshoot this?**

**Answer:**

```bash
kubectl describe pod <pod-name>
# Look at the Events section at the bottom
```

**All possible causes:**

| Cause | Event message | Fix |
|-------|--------------|-----|
| Insufficient CPU/memory on all nodes | `Insufficient cpu` or `Insufficient memory` | Add nodes, reduce resource requests, or delete other workloads |
| No node matches nodeSelector | `node(s) didn't match Pod's node affinity/selector` | Fix the nodeSelector labels or label the target node |
| All nodes are tainted | `node(s) had taints that the pod didn't tolerate` | Add a toleration to the Pod or untaint a node |
| PVC not bound | `persistentvolumeclaim "X" not found` or PVC stuck in Pending | Create the PV, fix the StorageClass, or check the cloud provider |
| ResourceQuota exceeded | `exceeded quota` | Increase the namespace quota or reduce resource requests |
| Too many Pods on node | `Too many pods` | Increase `--max-pods` on the kubelet or add nodes |
| Scheduler not running | No events at all | Check `kube-scheduler` Pod in `kube-system` |
| Pod affinity cannot be satisfied | `didn't match pod affinity rules` | Fix the affinity rules or ensure required Pods exist |

---

### Scenario 3: Pod stuck in ImagePullBackOff

**Situation:** A Pod is in `ImagePullBackOff` or `ErrImagePull`.

**How would you troubleshoot this?**

**Answer:**

```bash
kubectl describe pod <pod-name>
# Check Events for the exact pull error
```

**All possible causes:**

| Cause | Event message | Fix |
|-------|--------------|-----|
| Wrong image name or tag | `manifest unknown` or `not found` | Fix the image name/tag in the Pod spec |
| Private registry, no credentials | `unauthorized` or `authentication required` | Create an `imagePullSecret` and reference it in the Pod spec |
| Registry unreachable | `dial tcp: lookup ... no such host` or timeout | Check network connectivity, DNS, and firewall rules |
| Exceeded Docker Hub rate limit | `toomanyrequests` | Use an `imagePullSecret` with a paid account or mirror images |
| Image platform mismatch | `no matching manifest for linux/amd64` | Build the image for the correct architecture |

**Creating and using an image pull secret:**

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

---

### Scenario 4: Pod stuck in ContainerCreating

**Situation:** A Pod is stuck in `ContainerCreating` for a long time.

**How would you troubleshoot this?**

**Answer:**

```bash
kubectl describe pod <pod-name>
```

**All possible causes:**

| Cause | Event message | Fix |
|-------|--------------|-----|
| Volume mount failing | `Unable to attach or mount volumes` | Check PV/PVC status, StorageClass, and cloud disk availability |
| ConfigMap or Secret not found | `configmaps "X" not found` or `secrets "X" not found` | Create the missing ConfigMap or Secret |
| CNI plugin issue | `network plugin is not ready` | Check the CNI plugin DaemonSet in `kube-system` |
| Image pull taking too long | Pull in progress (large image) | Wait, or use a smaller image |
| Node disk pressure | `DiskPressure` condition on node | Free disk space on the node or add storage |

---

### Scenario 5: Pod is Running but application is not working

**Situation:** Pod shows `Running` and `1/1 Ready`, but the application does not respond.

**How would you troubleshoot this?**

**Answer:**

```bash
# Check logs
kubectl logs <pod-name>

# Exec into the container
kubectl exec -it <pod-name> -- /bin/sh

# Check if the process is running
kubectl exec <pod-name> -- ps aux

# Check if it is listening
kubectl exec <pod-name> -- ss -tlnp
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| App listening on wrong interface | `ss -tlnp` shows `127.0.0.1` instead of `0.0.0.0` | Configure the app to listen on `0.0.0.0` |
| App listening on wrong port | `ss -tlnp` shows a different port | Fix the port in the app config or the Pod spec |
| App deadlocked or hung | Process exists but no log output | Debug the app; add a liveness probe to auto-restart |
| Readiness probe misconfigured | Pod is `Running` but `0/1 Ready`; endpoint is missing from Service | Fix the readiness probe path or port |
| Missing config or env vars | Error in logs but app does not crash | Provide the correct ConfigMap/Secret/env vars |
| Dependency down | Timeout or connection errors in logs | Check the downstream service (database, API, cache) |

---

## Service and Networking Issues

### Scenario 6: Service not reachable from inside the cluster

**Situation:** You deployed an app and a Service, but curl from another Pod returns connection refused.

**How would you debug this?**

**Answer:**

**Step-by-step:**

```bash
# 1. Check the Service exists
kubectl get svc <service-name>

# 2. Check endpoints -- are there any backing Pods?
kubectl get endpoints <service-name>

# 3. If endpoints are empty, compare selector and labels
kubectl get svc <service-name> -o yaml | grep -A5 selector
kubectl get pods --show-labels

# 4. Test from a debug Pod
kubectl run debug --rm -it --image=busybox -- wget -qO- http://<service-name>:<port>

# 5. Test directly to the Pod IP
kubectl get pods -o wide
kubectl run debug --rm -it --image=busybox -- wget -qO- http://<pod-ip>:<container-port>
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| Selector does not match Pod labels | `kubectl get endpoints` is empty | Fix the selector in the Service or labels on the Pod |
| Wrong `targetPort` | Endpoints exist but connection refused | Set `targetPort` to the port the container actually listens on |
| Pods are not Ready | Endpoints list is empty even though Pods exist | Fix the readiness probe; Pods must pass readiness to get endpoints |
| Service is in wrong namespace | `kubectl get svc -n <namespace>` not found | Use the FQDN: `<service>.<namespace>.svc.cluster.local` |
| DNS not resolving | `nslookup <service>` fails | Check CoreDNS Pods in `kube-system`; restart if needed |
| Network policy blocking traffic | Direct Pod IP works but Service name does not, or both fail | Check NetworkPolicies in the namespace; add an ingress allow rule |

---

### Scenario 7: Service not reachable from outside the cluster

**Situation:** You have a NodePort or LoadBalancer Service, but external clients cannot reach it.

**How would you debug this?**

**Answer:**

```bash
kubectl get svc <service-name>
# Check the EXTERNAL-IP (for LoadBalancer) or NodePort
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| LoadBalancer stuck in `<pending>` | EXTERNAL-IP shows `<pending>` | Check cloud provider; ensure a load balancer controller is running |
| Firewall blocking NodePort range | `curl <node-ip>:<nodeport>` times out | Open ports 30000-32767 in the security group / firewall |
| Node security groups block traffic | Timeout from external network | Add inbound rule for the NodePort or LB port |
| Wrong NodePort port | Connecting to wrong port | Check `kubectl get svc` for the assigned NodePort |
| `externalTrafficPolicy: Local` and no Pod on that node | Some nodes work, others do not | Set to `Cluster` or ensure Pods run on all nodes (DaemonSet) |
| Node has no public IP | Cannot reach node from external network | Use a LoadBalancer Service or add a public IP |

---

### Scenario 8: DNS resolution failing inside Pods

**Situation:** Pods cannot resolve Service names or external domain names.

**How would you debug this?**

**Answer:**

```bash
# Test from inside a Pod
kubectl run debug --rm -it --image=busybox -- nslookup kubernetes.default
kubectl run debug --rm -it --image=busybox -- nslookup google.com
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| CoreDNS Pods are down | `kubectl get pods -n kube-system -l k8s-app=kube-dns` shows not Running | Check CoreDNS logs; restart or redeploy CoreDNS |
| CoreDNS ConfigMap misconfigured | CoreDNS logs show errors | Fix the `coredns` ConfigMap in `kube-system` |
| Pod's `dnsPolicy` set wrong | `dnsPolicy: None` or `Default` without `dnsConfig` | Set `dnsPolicy: ClusterFirst` (the default) |
| Network policy blocking DNS | Internal names fail | Allow egress to `kube-dns` on port 53 (TCP and UDP) |
| Upstream DNS unreachable | Internal names resolve but external names do not | Check `/etc/resolv.conf` on the nodes; fix upstream DNS |

---

## Deployment and Scaling Issues

### Scenario 9: Deployment rollout stuck

**Situation:** You updated a Deployment image, but `kubectl rollout status` hangs and new Pods are not becoming Ready.

**How would you troubleshoot this?**

**Answer:**

```bash
kubectl rollout status deployment <name>
kubectl get pods
kubectl describe pod <new-pod-name>
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| New image does not exist | Pod in `ImagePullBackOff` | Fix the image tag |
| New version crashes on startup | Pod in `CrashLoopBackOff` | Check logs; rollback with `kubectl rollout undo` |
| Readiness probe failing | Pod is `Running` but `0/1 Ready` | Fix the readiness probe endpoint or increase timeout |
| Insufficient resources | New Pod stuck in `Pending` | Reduce requests, add nodes, or remove resource-heavy workloads |
| PDB blocking eviction | Old Pods cannot be terminated | Check PodDisruptionBudget; ensure `maxUnavailable` allows updates |
| `maxUnavailable: 0` and no room for surge | No new Pod can be created | Increase `maxSurge` or free up cluster resources |

**Quick rollback:**
```bash
kubectl rollout undo deployment <name>
```

---

### Scenario 10: HPA not scaling up

**Situation:** Traffic is high but the HPA keeps the replica count at the minimum.

**How would you troubleshoot this?**

**Answer:**

```bash
kubectl get hpa
kubectl describe hpa <name>
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| Metrics Server not installed | `TARGETS` column shows `<unknown>` | Install Metrics Server |
| Pods have no resource requests | `<unknown>` in HPA targets | Add `resources.requests.cpu` (and/or memory) to the Pod spec |
| Metrics below threshold | HPA `describe` shows current < target | Traffic is not high enough; HPA is working correctly |
| HPA target is a percentage, requests are too high | CPU utilization looks low because requests are over-provisioned | Lower the CPU request to reflect actual baseline usage |
| `spec.replicas` set in Deployment | Deployment keeps resetting replicas | Remove `spec.replicas` from the Deployment manifest |
| Cooldown period | HPA recently scaled down | Wait for the stabilization window (default 5 min for scale-down) |

---

### Scenario 11: Node is NotReady

**Situation:** `kubectl get nodes` shows a node in `NotReady` state.

**How would you troubleshoot this?**

**Answer:**

```bash
kubectl describe node <node-name>
# Check the Conditions section
```

**All possible causes:**

| Cause | Condition shown | Fix |
|-------|----------------|-----|
| kubelet is down | No heartbeats; `Ready=False` | SSH into the node; check `systemctl status kubelet` and restart |
| kubelet cannot reach API server | `Ready=Unknown` | Check network connectivity between node and control plane |
| Node ran out of disk | `DiskPressure=True` | Free disk space; clean up unused images with `crictl rmi --prune` |
| Node ran out of memory | `MemoryPressure=True` | Identify memory-heavy processes; add swap or memory |
| Too many PIDs | `PIDPressure=True` | Identify processes forking excessively |
| Container runtime crashed | kubelet logs show CRI errors | Check `systemctl status containerd` (or CRI-O); restart |
| Certificate expired | kubelet logs show TLS errors | Renew kubelet certificates; `kubeadm certs renew` |
| Node network unreachable | SSH fails | Check cloud instance status; reboot if needed |

---

## Storage Issues

### Scenario 12: PVC stuck in Pending

**Situation:** A PersistentVolumeClaim stays in `Pending` and the Pod that uses it is also stuck.

**How would you troubleshoot this?**

**Answer:**

```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
```

**All possible causes:**

| Cause | Event message | Fix |
|-------|--------------|-----|
| No matching PV available | `no persistent volumes available` | Create a PV that matches the PVC's size, access mode, and StorageClass |
| StorageClass does not exist | `storageclass.storage.k8s.io "X" not found` | Create the StorageClass or fix the name in the PVC |
| StorageClass has no provisioner | PVC pending with no events | Ensure the provisioner Pod is running (e.g., EBS CSI driver) |
| Requested size exceeds available storage | Cloud provider error in events | Reduce the requested size or increase cloud quotas |
| Access mode mismatch | `ReadWriteMany` requested but PV only supports `ReadWriteOnce` | Change the access mode or use a storage backend that supports RWX (NFS, EFS) |
| PV is in a different zone | Pod describes shows zone mismatch | Use `volumeBindingMode: WaitForFirstConsumer` in the StorageClass |

---

### Scenario 13: Data lost after Pod restart

**Situation:** A database Pod loses all its data every time it restarts.

**Why is this happening?**

**Answer:**

Container filesystems are ephemeral. When a container restarts, its writable layer is reset.

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| No volume mounted | `kubectl get pod -o yaml` shows no volumes | Add a PVC and mount it at the data directory |
| Volume mounted at wrong path | Data directory is not where the volume is mounted | Check the database docs for the correct data path (e.g., `/var/lib/postgresql/data`) |
| Using `emptyDir` | Volume is `emptyDir` -- deleted when Pod is removed | Switch to a PVC with a PersistentVolume |
| PV `reclaimPolicy: Delete` | PV is deleted when PVC is deleted | Use `reclaimPolicy: Retain` for important data |
| StatefulSet deleted with cascade | PVCs were deleted along with the StatefulSet | Use `kubectl delete statefulset <name> --cascade=orphan` to keep PVCs |

---

## Security and RBAC Issues

### Scenario 14: Permission denied when running kubectl commands

**Situation:** A developer runs `kubectl get pods` and gets `Error from server (Forbidden)`.

**How would you fix this?**

**Answer:**

```bash
# Check what the user can do
kubectl auth can-i get pods --as=<username>
kubectl auth can-i --list --as=<username>
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| No Role/ClusterRole bound | `auth can-i` returns `no` | Create a Role and RoleBinding for the user |
| RoleBinding in wrong namespace | User can access one namespace but not another | Create a RoleBinding in the correct namespace |
| ServiceAccount token missing | Pod app gets 403 from API server | Add a ServiceAccount with the correct Role |
| ClusterRole needed but only Role exists | Works in one namespace, fails in others or on cluster-scoped resources | Use ClusterRole and ClusterRoleBinding |
| Wrong group or user in the binding | Binding exists but user claim does not match | Check the `subjects` in the RoleBinding; fix user/group name |

**Example fix -- grant read access to Pods in a namespace:**

```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create rolebinding dev-pod-reader --role=pod-reader --user=jane -n dev
```

---

### Scenario 15: Pod cannot access the Kubernetes API

**Situation:** An application Pod tries to call the Kubernetes API but gets `403 Forbidden` or connection errors.

**How would you fix this?**

**Answer:**

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| Default ServiceAccount has no permissions | Pod uses `default` SA which has no RBAC | Create a dedicated ServiceAccount with a proper Role |
| `automountServiceAccountToken: false` | No token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/` | Set `automountServiceAccountToken: true` or mount manually |
| NetworkPolicy blocking access to API server | Connection timeout to `kubernetes.default.svc` | Allow egress to the API server IP on port 443 |
| API server URL misconfigured | App is not using `kubernetes.default.svc` | Use the in-cluster config (every SDK supports it) |

---

## Configuration and Scheduling Issues

### Scenario 16: ConfigMap or Secret changes not reflected in Pods

**Situation:** You updated a ConfigMap but the running Pods still use the old values.

**Why is this happening?**

**Answer:**

**All possible causes:**

| Cause | Behavior | Fix |
|-------|----------|-----|
| ConfigMap is injected as env vars | Env vars are set at Pod creation and never updated | Restart the Pods: `kubectl rollout restart deployment <name>` |
| ConfigMap is mounted as volume but cache has not refreshed | Takes up to 60-90 seconds for the kubelet to sync | Wait for the sync interval; or restart the Pod |
| App does not reload config files | The file on disk changes but the app read it only at startup | Add a config reload mechanism or sidecar that watches for changes |
| Immutable ConfigMap | `kubectl edit configmap` fails | Delete and recreate the ConfigMap; restart Pods |

**Best practice to force rollout on config change:** Use a hash annotation so the Deployment detects the change:

```bash
kubectl rollout restart deployment <name>
```

Or in Helm/Kustomize, add an annotation with the ConfigMap checksum to the Pod template so any change triggers a new rollout.

---

### Scenario 17: Pod scheduled on wrong node

**Situation:** A Pod keeps getting scheduled on a node you do not want it on.

**How would you fix this?**

**Answer:**

**Possible causes and fixes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| No nodeSelector or affinity set | Pod has no scheduling constraints | Add `nodeSelector` or `nodeAffinity` to target correct nodes |
| Node missing the expected label | `kubectl get nodes --show-labels` | Label the node: `kubectl label node <node> key=value` |
| Taint on desired node | Pod goes elsewhere because it does not tolerate the taint | Add a toleration to the Pod spec |
| Pod anti-affinity preventing colocation | Already a Pod with the same label on the desired node | Adjust the anti-affinity rule or the labels |
| Topology spread constraint | Scheduler distributing evenly across zones | Adjust `topologySpreadConstraints` |

---

### Scenario 18: Pod evicted

**Situation:** Pods are being evicted unexpectedly.

**What are the possible causes?**

**Answer:**

```bash
kubectl describe pod <pod-name>
# Look for eviction reason in Events or Status
```

**All possible causes:**

| Cause | Eviction reason | Fix |
|-------|----------------|-----|
| Node disk pressure | `DiskPressure` | Free disk; clean images: `crictl rmi --prune` |
| Node memory pressure | `MemoryPressure` | Reduce pod memory usage; add memory to node |
| Pod exceeded memory limit | `OOMKilled` | Increase `resources.limits.memory` |
| ResourceQuota exceeded | `exceeded quota` | Increase quota or reduce resource usage |
| Node drain (maintenance) | `Evicted` by drain | Expected behavior; Pod will be rescheduled |
| Priority preemption | Higher-priority Pod needs room | Set appropriate PriorityClass on critical workloads |
| Ephemeral storage exceeded | `ephemeral-storage exceeded` | Increase ephemeral storage limit or reduce log/temp file output |

**QoS classes and eviction order:**

| QoS Class | Evicted first? | Condition |
|-----------|---------------|-----------|
| BestEffort | Yes (first) | No requests or limits set |
| Burstable | Second | Requests set but lower than limits |
| Guaranteed | Last | Requests equal limits for all containers |

Set `requests = limits` on critical Pods to get `Guaranteed` QoS and be the last to be evicted.

---

## Cluster and Control Plane Issues

### Scenario 19: kubectl commands timing out

**Situation:** `kubectl get pods` hangs or returns a timeout error.

**How would you troubleshoot this?**

**Answer:**

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| API server is down | Cannot connect to cluster | Check `kube-apiserver` static Pod on control plane node |
| Wrong kubeconfig | `kubectl config view` shows wrong server URL | Fix `~/.kube/config` or set `KUBECONFIG` env var |
| Network between client and API server | `curl -k https://<api-server>:6443/healthz` fails | Check VPN, firewall, security groups |
| API server overloaded | Partial responses, high latency | Check API server metrics; scale control plane |
| etcd is unhealthy | API server logs show etcd connection errors | Check etcd Pod or process; verify disk performance |
| Client certificate expired | `x509: certificate has expired` | Renew kubeconfig certs: `kubeadm kubeconfig user` |

---

### Scenario 20: etcd cluster unhealthy

**Situation:** The cluster is behaving erratically. API responses are slow or inconsistent.

**How would you troubleshoot etcd?**

**Answer:**

```bash
# Check etcd Pod
kubectl get pods -n kube-system -l component=etcd

# Check etcd health (from control plane node)
etcdctl endpoint health --cluster
etcdctl endpoint status --cluster --write-out=table
```

**All possible causes:**

| Cause | Symptoms | Fix |
|-------|----------|-----|
| Slow disk I/O | High etcd latency, leader elections | Use SSD for etcd data directory |
| etcd member down | Cluster has less than quorum | Restart the etcd member or restore from backup |
| Database too large | `etcdserver: mvcc: database space exceeded` | Compact and defragment: `etcdctl compact` + `etcdctl defrag` |
| Clock skew between nodes | Leader election issues | Sync clocks with NTP/chrony |
| Network partition | Split brain symptoms | Fix network between control plane nodes |

---

## Multi-Container and Observability Issues

### Scenario 21: Liveness probe kills healthy Pods

**Situation:** Pods keep restarting even though the application is working fine.

**How would you fix this?**

**Answer:**

```bash
kubectl describe pod <pod-name>
# Check Events for "Liveness probe failed" and restart count
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| Probe endpoint returns non-200 | Logs show the probe path is not handled | Fix the probe path to match an actual health endpoint |
| Probe timeout too short | App occasionally takes longer than `timeoutSeconds` | Increase `timeoutSeconds` |
| Probe starts before app is ready | Restarts happen right after startup | Add a `startupProbe` or increase `initialDelaySeconds` |
| Probe port wrong | Connection refused in events | Fix the probe port to match `containerPort` |
| App health endpoint depends on external service | DB down causes health check failure | Make health endpoint check only the app itself, not dependencies (use readiness for dependency checks) |
| Resource starvation (CPU limit throttling) | Probe times out under load | Increase CPU limits or use `burstable` QoS |

**Recommended probe setup:**

```yaml
startupProbe:          # gives the app time to start
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 2

livenessProbe:         # restarts if app is dead
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10
  timeoutSeconds: 3

readinessProbe:        # removes from Service if not ready
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
```

---

### Scenario 22: Cannot exec into or get logs from a Pod

**Situation:** `kubectl exec` or `kubectl logs` returns errors.

**All possible causes:**

| Cause | Error message | Fix |
|-------|--------------|-----|
| Container has no shell | `OCI runtime exec failed: exec: "bash": executable file not found` | Use `sh` instead of `bash`; for distroless, use `kubectl debug` |
| Pod has already terminated | `container not found` | Use `--previous` for logs; check the new Pod |
| kubelet on the node is down | `error dialing backend` | Check kubelet status on the node |
| RBAC does not allow exec | `Forbidden` | Add `pods/exec` and `pods/log` permissions to the Role |
| API server cannot reach the node | Timeout | Check network path between control plane and worker node |

**Debugging distroless containers (no shell):**

```bash
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
```

This attaches an ephemeral container with debugging tools to the running Pod.

---

### Scenario 23: Ingress not routing traffic

**Situation:** You created an Ingress resource but HTTP requests to the domain return 404 or connection refused.

**How would you troubleshoot this?**

**Answer:**

```bash
kubectl get ingress
kubectl describe ingress <name>
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| No Ingress controller installed | No events on the Ingress resource | Install an Ingress controller (nginx-ingress, Traefik, etc.) |
| Wrong `ingressClassName` | Ingress exists but controller ignores it | Set `ingressClassName` to match the controller |
| Backend Service does not exist | Describe shows `service "X" not found` | Create the Service or fix the name |
| Backend Service has no endpoints | Service exists but Pods are not ready | Fix Pod readiness; check selector match |
| TLS secret missing | `kubectl describe` shows TLS error | Create the TLS Secret referenced in the Ingress |
| Host header mismatch | Curl without Host header gets default backend | Send the correct Host header or use `curl -H "Host: app.example.com"` |
| Path matching issue | Some paths work, others do not | Check `pathType` -- use `Prefix` for catch-all or `Exact` for specific paths |

---

### Scenario 24: Job or CronJob not running

**Situation:** A CronJob is not creating Jobs on schedule, or a Job Pod keeps failing.

**How would you troubleshoot this?**

**Answer:**

```bash
kubectl get cronjobs
kubectl describe cronjob <name>
kubectl get jobs
kubectl get pods --selector=job-name=<job-name>
```

**All possible causes:**

| Cause | How to confirm | Fix |
|-------|---------------|-----|
| Cron schedule syntax wrong | CronJob exists but no Jobs created | Fix the `schedule` field (use crontab.guru to validate) |
| `suspend: true` | CronJob is suspended | Set `suspend: false` |
| `concurrencyPolicy: Forbid` and previous Job still running | New Jobs are skipped | Wait for the old Job to finish or set `concurrencyPolicy: Replace` |
| Job Pod crashes | Pods in `CrashLoopBackOff` or `Error` | Check Pod logs; fix the command or image |
| `backoffLimit` exhausted | Job shows `Failed` | Increase `backoffLimit` or fix the underlying error |
| `activeDeadlineSeconds` exceeded | Job terminated early | Increase the deadline or optimize the task |
| `startingDeadlineSeconds` missed | CronJob missed its window (controller was down) | Increase `startingDeadlineSeconds` |
| `successfulJobsHistoryLimit: 0` | Cannot see completed Jobs | Increase the limit to debug; check logs before cleanup |

---
