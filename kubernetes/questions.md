# Kubernetes -- Interview Questions

General and conceptual interview questions with clear answers.

---

### Q1: What is Kubernetes and why is it needed?

**Answer:**
Kubernetes (k8s) is an open-source container orchestration platform. It automates deployment, scaling, and management of containerized applications.

It is needed because running containers manually does not scale. Kubernetes handles:

- Service discovery and load balancing
- Automated rollouts and rollbacks
- Self-healing (restarting failed containers)
- Horizontal scaling
- Secret and configuration management

---

### Q2: Explain the Kubernetes architecture.

**Answer:**

**Control Plane:**

- `kube-apiserver` -- Front-end for the cluster; all communication goes through it.
- `etcd` -- Key-value store holding all cluster state.
- `kube-scheduler` -- Assigns pods to nodes based on resource availability and constraints.
- `kube-controller-manager` -- Runs controllers (Deployment, ReplicaSet, Node, etc.) that reconcile desired state with actual state.

**Worker Nodes:**

- `kubelet` -- Agent on each node that ensures containers are running as expected.
- `kube-proxy` -- Manages network rules for Service communication.
- Container runtime (e.g., containerd, CRI-O) -- Pulls images and runs containers.

---

### Q3: Describe the lifecycle of a resource creation request in Kubernetes.

**Answer:**

When you run `kubectl apply -f deployment.yaml`, the following sequence occurs:

**1. kubectl validates and sends the request**
- `kubectl` validates the manifest locally (schema check).
- It sends an HTTP POST/PUT request to the `kube-apiserver` on the control plane.

**2. API server authenticates and authorizes**
- The API server verifies the caller's identity (authentication via certificates, tokens, or OIDC).
- It checks if the caller is allowed to perform this action (authorization via RBAC).

**3. Admission controllers process the request**
- The request passes through admission controllers (mutating and validating).
- Mutating controllers can modify the object (e.g., inject default values, add sidecar containers).
- Validating controllers can reject the request (e.g., enforce policies like resource limits).

**4. Object is persisted to etcd**
- The API server writes the desired state (the Deployment object) to `etcd`.
- At this point, the resource exists in the cluster's desired state but nothing is running yet.

**5. Controllers detect the new object**
- The Deployment controller watches for changes. It sees the new Deployment and creates a ReplicaSet.
- The ReplicaSet controller sees the new ReplicaSet and creates Pod objects to match the desired replica count.

**6. Scheduler assigns Pods to nodes**
- The `kube-scheduler` watches for unscheduled Pods (Pods with no `nodeName`).
- It evaluates node resources, affinity rules, taints/tolerations, and selects the best node.
- It updates the Pod's `nodeName` field in `etcd` via the API server.

**7. kubelet starts the containers**
- The `kubelet` on the assigned node watches for Pods scheduled to its node.
- It instructs the container runtime (containerd, CRI-O) to pull the image and start the container.
- It sets up networking (via CNI plugin) and mounts volumes.

**8. kube-proxy updates network rules**
- If a Service exists for these Pods, `kube-proxy` updates iptables/IPVS rules so traffic can reach the new Pods.

**9. Readiness and liveness probes begin**
- The `kubelet` starts executing any configured readiness and liveness probes.
- The Pod is only added to Service endpoints once the readiness probe passes.

**Summary of the flow:**

```
kubectl --> API server --> Admission Controllers --> etcd
                                                      |
            kubelet <-- Scheduler <-- Controllers <---+
              |
          Container Runtime --> Container Running
```

---

### Q4: Explain the Node, ReplicaSet, and Deployment controllers and how they interact with the API server.

**Answer:**

All controllers in Kubernetes follow the same pattern: they **watch** the API server for changes, compare the **desired state** (what is declared) with the **actual state** (what is running), and take action to reconcile any differences. They never talk to each other directly -- all communication goes through the API server and `etcd`.

---

**Node Controller**

Responsible for monitoring the health of nodes in the cluster.

- Watches the API server for Node objects.
- Each node's `kubelet` sends heartbeats to the API server at regular intervals.
- If the Node controller detects that heartbeats have stopped:
  - It waits for a grace period (`--node-monitor-grace-period`, default 40s).
  - It marks the node's condition as `NotReady`.
  - After a timeout (`--pod-eviction-timeout`, default 5m), it evicts all Pods from the unreachable node by creating eviction requests through the API server.
- It also assigns CIDR blocks to new nodes (if configured).

```
Node Controller --watch--> API Server (Node objects)
                --update-> API Server (set condition to NotReady)
                --create-> API Server (Pod eviction requests)
```

---

**ReplicaSet Controller**

Ensures that the correct number of Pod replicas are running at all times.

- Watches the API server for ReplicaSet objects and Pod objects.
- Compares `spec.replicas` (desired) with the count of Pods matching the ReplicaSet's label selector (actual).
- If there are fewer Pods than declared, it creates new Pod objects through the API server.
- If there are more Pods than declared, it deletes excess Pods through the API server.
- It does not restart or replace unhealthy Pods -- it only cares about the count.

```
ReplicaSet Controller --watch--> API Server (ReplicaSet + Pod objects)
                      --create-> API Server (new Pod objects)
                      --delete-> API Server (excess Pod objects)
```

---

**Deployment Controller**

Manages ReplicaSets to enable declarative updates, rollouts, and rollbacks.

- Watches the API server for Deployment objects.
- When a Deployment is created or updated, the Deployment controller creates a **new ReplicaSet** with the updated Pod template.
- It scales up the new ReplicaSet and scales down the old one according to the rollout strategy:
  - **RollingUpdate** (default): gradually replaces old Pods with new ones, respecting `maxSurge` and `maxUnavailable`.
  - **Recreate**: kills all old Pods before creating new ones.
- It keeps old ReplicaSets around (controlled by `revisionHistoryLimit`) to enable rollbacks.
- On `kubectl rollout undo`, it scales a previous ReplicaSet back up.

```
Deployment Controller --watch---> API Server (Deployment objects)
                      --create--> API Server (new ReplicaSet)
                      --update--> API Server (scale old/new ReplicaSets)
```

---

**How they work together (example: creating a Deployment with 3 replicas):**

```
1. User: kubectl apply -f deployment.yaml
                    |
2. API Server: stores Deployment in etcd
                    |
3. Deployment Controller: sees new Deployment
   --> creates ReplicaSet (replicas=3) via API Server
                    |
4. ReplicaSet Controller: sees new ReplicaSet
   --> creates 3 Pod objects via API Server
                    |
5. Scheduler: assigns each Pod to a node via API Server
                    |
6. kubelet: starts containers on each assigned node
                    |
7. Node Controller: monitors node health continuously
```

The key insight is that each controller has a **single responsibility** and only interacts with the API server. This loose coupling is what makes Kubernetes extensible -- you can add custom controllers that follow the exact same watch-reconcile pattern.

---

### Q5: How does the Kubernetes scheduler work?

**Answer:**

The `kube-scheduler` watches the API server for Pods that have no `nodeName` assigned. For each unscheduled Pod, it runs a two-phase process:

**Phase 1 -- Filtering (predicates)**

Eliminates nodes that cannot run the Pod. A node is filtered out if:

- It does not have enough CPU or memory to satisfy the Pod's resource requests.
- It has a taint that the Pod does not tolerate.
- The Pod has a `nodeSelector` or `nodeAffinity` that the node does not match.
- A `PodAntiAffinity` rule would be violated.
- The node is marked `Unschedulable` (cordoned).

**Phase 2 -- Scoring (priorities)**

Ranks the remaining nodes to find the best fit. Scoring factors include:

- **LeastRequestedPriority** -- Prefer nodes with the most available resources.
- **BalancedResourceAllocation** -- Prefer nodes where CPU and memory usage are balanced.
- **NodeAffinity** -- Higher score for nodes matching preferred (not required) affinity rules.
- **Pod topology spread** -- Distribute Pods evenly across zones or nodes.

The node with the highest total score wins. The scheduler updates the Pod's `nodeName` via the API server, and the `kubelet` on that node picks it up.

**Key scheduling features:**

- **Taints and tolerations** -- Nodes repel Pods unless the Pod explicitly tolerates the taint.
- **Node affinity** -- `requiredDuringScheduling` (hard rule) vs `preferredDuringScheduling` (soft preference).
- **Pod affinity / anti-affinity** -- Co-locate or separate Pods based on labels of already-running Pods.
- **Resource requests vs limits** -- The scheduler uses `requests` (not limits) to decide placement.

---

### Q6: What does the kubelet do?

**Answer:**

The `kubelet` is the primary agent that runs on every worker node. It is responsible for making sure that containers described in Pod specs are running and healthy on its node.

**Core responsibilities:**

- **Watches the API server** for Pods assigned to its node (by checking `spec.nodeName`).
- **Instructs the container runtime** (containerd, CRI-O) to pull images, create containers, and manage their lifecycle via the CRI (Container Runtime Interface).
- **Runs probes** on containers:
  - `livenessProbe` -- Restarts the container if it fails.
  - `readinessProbe` -- Removes the Pod from Service endpoints if it fails.
  - `startupProbe` -- Blocks liveness/readiness checks until the container finishes starting.
- **Reports status** back to the API server: Pod phase, container states, resource usage, and node conditions.
- **Manages volumes** -- Mounts ConfigMaps, Secrets, PersistentVolumes, and projected volumes into containers.
- **Sends heartbeats** -- Periodically updates the Node's Lease object so the control plane knows the node is alive.
- **Handles Pod eviction** -- If the node runs low on memory or disk, the kubelet evicts lower-priority Pods based on QoS class (BestEffort first, then Burstable, then Guaranteed).

The kubelet does **not** manage containers that were not created by Kubernetes. It only cares about Pods assigned to its node through the API server.

---

### Q7: What is kube-proxy, what is a CNI plugin, and how do they differ?

**Answer:**

**kube-proxy**

A network component that runs on every node and implements Kubernetes **Service** networking. When a Pod sends traffic to a Service's ClusterIP, kube-proxy routes it to one of the backing Pods.

It does this using one of three modes:
- **iptables** (default) -- Creates iptables rules that DNAT traffic from the Service IP to a randomly selected Pod IP.
- **IPVS** -- Uses Linux IPVS for load balancing. Better performance at scale with support for multiple balancing algorithms (round-robin, least connections, etc.).
- **userspace** -- Legacy mode. Proxies traffic through a userspace process. Rarely used.

**CNI (Container Network Interface)**

A plugin that the kubelet calls when a Pod is created or deleted. It is responsible for assigning an IP address to the Pod and connecting it to the cluster network so Pods can communicate with each other.

Common CNI plugins:
- **Calico** -- L3 networking with BGP, supports network policies.
- **Flannel** -- Simple overlay network using VXLAN.
- **Cilium** -- eBPF-based, high performance, advanced network policies and observability.
- **Weave Net** -- Mesh overlay network.

**How they differ:**

| Aspect | kube-proxy | CNI plugin |
|--------|-----------|------------|
| Scope | Service-to-Pod routing | Pod-to-Pod networking |
| What it manages | ClusterIP, NodePort, LoadBalancer traffic | Pod IP assignment and network connectivity |
| When it acts | When Service or Endpoints objects change | When a Pod is created or deleted |
| Operates at | iptables / IPVS rules | Network interfaces, routes, overlays |

In short: CNI gives each Pod an IP and network connectivity. kube-proxy makes Services work by routing traffic to the right Pod.

---

## Core Resources

### Q8: What is a Pod and what can it contain?

**Answer:**

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share:

- The same network namespace (same IP, communicate via `localhost`)
- The same storage volumes
- The same lifecycle (created and deleted together)

**Single-container Pod** -- The most common pattern. One application per Pod.

**Multi-container Pod** -- Used for sidecar patterns:
- **Sidecar** -- Adds functionality (e.g., log shipper, proxy like Envoy)
- **Init container** -- Runs before the main container starts (e.g., wait for a database, run migrations)
- **Ephemeral container** -- Injected into a running Pod for debugging (`kubectl debug`)

Example Pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

You rarely create Pods directly. You use a Deployment, which manages Pods through ReplicaSets.

---

### Q9: What is a Deployment and how do rollout strategies work?

**Answer:**

A Deployment is a declarative way to manage a set of identical Pods. It creates and manages ReplicaSets, which in turn manage Pods.

**What it provides:**
- Desired replica count
- Rolling updates when the Pod template changes
- Automatic rollback on failure
- Revision history

**Rollout strategies:**

**RollingUpdate (default):**
- Gradually replaces old Pods with new ones.
- `maxSurge` -- How many extra Pods can exist during the update (default 25%).
- `maxUnavailable` -- How many Pods can be unavailable during the update (default 25%).

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0    # zero-downtime: always keep all replicas available
```

**Recreate:**
- Kills all old Pods before creating new ones.
- Causes downtime. Used when the application cannot run two versions simultaneously (e.g., database schema conflicts).

```yaml
spec:
  strategy:
    type: Recreate
```

**Useful commands:**

```bash
kubectl rollout status deployment myapp
kubectl rollout history deployment myapp
kubectl rollout undo deployment myapp                 # rollback to previous
kubectl rollout undo deployment myapp --to-revision=2 # rollback to specific revision
```

---

### Q10: What are Services and what are the different types?

**Answer:**

A Service provides a stable network endpoint to access a set of Pods. Pods are ephemeral and get new IPs when recreated, so Services give a constant IP and DNS name.

A Service selects Pods using label selectors and load-balances traffic across them.

**Service types:**

**ClusterIP (default):**
- Internal-only. Accessible only from within the cluster.
- Gets a virtual IP from the cluster's service CIDR.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

**NodePort:**
- Exposes the Service on a static port (30000-32767) on every node's IP.
- External traffic can reach the Service via `<NodeIP>:<NodePort>`.
- Also creates a ClusterIP automatically.

**LoadBalancer:**
- Provisions an external load balancer from the cloud provider (AWS ELB, GCP LB, etc.).
- External traffic hits the LB, which forwards to NodePorts, which forward to Pods.
- Also creates a NodePort and ClusterIP automatically.

**ExternalName:**
- Maps the Service to a DNS CNAME record (e.g., `mydb.example.com`).
- No proxying, no ClusterIP. Just a DNS alias.

**Headless Service** (`clusterIP: None`):
- No virtual IP. DNS returns the individual Pod IPs directly.
- Used for StatefulSets where clients need to address specific Pods.

---

### Q11: What are ConfigMaps and how are they used?

**Answer:**

A ConfigMap stores non-sensitive configuration data as key-value pairs. It decouples configuration from the container image so the same image can be used across environments.

**Creating a ConfigMap:**

```bash
# From literal values
kubectl create configmap app-config --from-literal=LOG_LEVEL=info --from-literal=PORT=8080

# From a file
kubectl create configmap nginx-config --from-file=nginx.conf
```

**Using a ConfigMap in a Pod:**

As environment variables:
```yaml
envFrom:
  - configMapRef:
      name: app-config
```

As a mounted volume (useful for config files):
```yaml
volumes:
  - name: config
    configMap:
      name: nginx-config
volumeMounts:
  - name: config
    mountPath: /etc/nginx/nginx.conf
    subPath: nginx.conf
```

**Key points:**
- ConfigMaps are not encrypted. Do not put secrets in them.
- Changes to a mounted ConfigMap are reflected in the Pod automatically (after a short delay), but environment variables require a Pod restart.
- Maximum size is 1 MiB.

---

### Q12: What are Secrets and how do they differ from ConfigMaps?

**Answer:**

Secrets store sensitive data such as passwords, tokens, and TLS certificates. They are functionally similar to ConfigMaps but with a few differences:

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| Purpose | Non-sensitive config | Sensitive data |
| Stored as | Plain text in etcd | Base64-encoded in etcd (not encrypted by default) |
| Size limit | 1 MiB | 1 MiB |
| tmpfs mount | No | Yes (mounted as tmpfs, never written to disk on the node) |

**Creating a Secret:**

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cret
```

**Using a Secret in a Pod:**

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: password
```

**Important notes:**
- Base64 encoding is **not** encryption. Anyone with access to `etcd` or the API can read them.
- Enable **encryption at rest** in the API server to encrypt Secrets in `etcd`.
- Use RBAC to restrict who can read Secrets.
- For production, consider external secret managers (HashiCorp Vault, AWS Secrets Manager, Sealed Secret Operator) with tools like External Secrets Operator.

---

### Q13: What are Namespaces and when should you use them?

**Answer:**

A Namespace is a logical partition within a cluster. It provides a scope for names -- two resources can have the same name as long as they are in different Namespaces.

**Default Namespaces:**

| Namespace | Purpose |
|-----------|---------|
| `default` | Where resources go if no namespace is specified |
| `kube-system` | Control plane components (API server, scheduler, CoreDNS, etc.) |
| `kube-public` | Readable by all users, used for cluster-wide public info |
| `kube-node-lease` | Node heartbeat Lease objects |

**When to use Namespaces:**
- Separate environments in a shared cluster (dev, staging, prod)
- Isolate teams or projects
- Apply per-namespace resource quotas and limit ranges
- Scope RBAC policies (e.g., team A can only deploy to `team-a` namespace)

**Common commands:**

```bash
kubectl get namespaces
kubectl create namespace dev
kubectl get pods -n dev
kubectl config set-context --current --namespace=dev  # set default namespace
```

**What is namespace-scoped vs cluster-scoped:**
- **Namespace-scoped** -- Pods, Services, Deployments, ConfigMaps, Secrets, Roles, RoleBindings
- **Cluster-scoped** -- Nodes, PersistentVolumes, Namespaces, ClusterRoles, ClusterRoleBindings

Namespaces do **not** provide network isolation by default. Pods across namespaces can communicate freely unless you apply NetworkPolicies.

---

### Q14: What are taints and tolerations?

**Answer:**

Taints and tolerations control which Pods can be scheduled on which nodes.

- A **taint** is applied to a node. It repels Pods that do not tolerate it.
- A **toleration** is applied to a Pod. It allows the Pod to be scheduled on a tainted node (but does not force it).

**Taint effects:**

| Effect | Behavior |
|--------|----------|
| `NoSchedule` | New Pods without a matching toleration will not be scheduled on this node. Existing Pods are unaffected. |
| `PreferNoSchedule` | The scheduler tries to avoid the node, but will use it if no other option exists. |
| `NoExecute` | New Pods are not scheduled, and existing Pods without a matching toleration are evicted. |

**Applying a taint:**

```bash
# Taint a node
kubectl taint nodes node1 gpu=true:NoSchedule

# Remove a taint (trailing minus)
kubectl taint nodes node1 gpu=true:NoSchedule-
```

**Adding a toleration to a Pod:**

```yaml
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

You can also use `operator: "Exists"` to match any value for that key:

```yaml
tolerations:
  - key: "gpu"
    operator: "Exists"
    effect: "NoSchedule"
```

**Common use cases:**
- **Dedicated nodes** -- Taint GPU nodes so only ML workloads with the right toleration run there.
- **Control plane isolation** -- Control plane nodes have a `node-role.kubernetes.io/control-plane:NoSchedule` taint by default so user Pods are not scheduled on them.
- **Node draining** -- `kubectl drain` applies a `NoExecute` taint to evict all Pods before maintenance.

Taints repel. Tolerations allow, but do not attract. To force a Pod onto a specific node, combine tolerations with node affinity.

---

### Q15: What are labels and selectors?

**Answer:**

**Labels** are key-value pairs attached to any Kubernetes object. They are used to organize and identify resources.

```yaml
metadata:
  labels:
    app: frontend
    env: production
    team: platform
```

**Selectors** query objects by their labels. They are how controllers and Services find the Pods they manage.

**Equality-based selectors:**
```bash
kubectl get pods -l app=frontend
kubectl get pods -l env!=staging
```

**Set-based selectors:**
```bash
kubectl get pods -l 'env in (production, staging)'
kubectl get pods -l 'team notin (legacy)'
kubectl get pods -l 'gpu'           # key exists, any value
kubectl get pods -l '!experimental' # key does not exist
```

**Where selectors are used:**
- **Services** -- Select which Pods receive traffic
- **Deployments / ReplicaSets** -- Select which Pods they own
- **Network Policies** -- Select which Pods the policy applies to
- **kube-scheduler** -- Node affinity and pod affinity use label selectors

Labels are the primary mechanism for loose coupling in Kubernetes. Objects do not reference each other by name; they find each other through labels.

---

### Q16: What is the difference between nodeSelector and node affinity?

**Answer:**

Both control which nodes a Pod can be scheduled on, but node affinity is more expressive.

**nodeSelector** -- Simple key-value match. The Pod runs only on nodes whose labels match exactly.

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

**Node affinity** -- Supports operators (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`) and has two strength levels:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # hard rule
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - eu-west-1a
                  - eu-west-1b
      preferredDuringSchedulingIgnoredDuringExecution:  # soft preference
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
```

**Comparison:**

| Feature | nodeSelector | Node affinity |
|---------|-------------|---------------|
| Syntax | Simple key=value | Expressive match expressions |
| Hard/soft rules | Hard only | `required` (hard) and `preferred` (soft) |
| Operators | Equality only | In, NotIn, Exists, DoesNotExist, Gt, Lt |
| Multiple conditions | AND only | AND within a term, OR across terms |
| Weighted preferences | No | Yes (via `weight` field) |

Use `nodeSelector` for simple cases. Use node affinity when you need soft preferences, set-based matching, or weighted scoring.

---

### Q17: What is a DaemonSet?

**Answer:**

A DaemonSet ensures that a copy of a Pod runs on every node in the cluster (or a subset of nodes if you use a node selector or affinity).

When a new node joins the cluster, a Pod is automatically created on it. When a node is removed, the Pod is garbage-collected.

**Use cases:**
- **Log collection** -- Run a Fluentd or Filebeat agent on every node
- **Monitoring** -- Run a node exporter or Datadog agent on every node
- **Networking** -- CNI plugins (Calico, Cilium) and kube-proxy run as DaemonSets
- **Storage** -- CSI node drivers

**Example:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          ports:
            - containerPort: 9100
```

DaemonSets respect taints and tolerations. To run on control plane nodes, the Pod must tolerate the `node-role.kubernetes.io/control-plane:NoSchedule` taint.

---

### Q18: What are static Pods?

**Answer:**

Static Pods are managed directly by the `kubelet` on a specific node, without the API server or any controller being involved.

The kubelet watches a local directory (default: `/etc/kubernetes/manifests/`) for Pod manifests. When a YAML file is placed there, the kubelet creates and manages that Pod. If the file is removed, the Pod is deleted.

**How they differ from regular Pods:**

| Aspect | Regular Pod | Static Pod |
|--------|-------------|------------|
| Managed by | API server + controllers | kubelet directly |
| Visible in API | Yes | Yes (as a mirror Pod, read-only) |
| Scheduling | kube-scheduler assigns node | Always on the node where the manifest lives |
| Self-healing | Controller recreates on failure | kubelet restarts on failure |
| Updates | Via `kubectl apply` | Edit the manifest file on the node |

**Primary use case:** The control plane itself. On clusters bootstrapped with `kubeadm`, the API server, controller manager, scheduler, and etcd all run as static Pods:

```bash
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

This solves the chicken-and-egg problem -- the kubelet can start control plane components before the API server exists.

---

### Q19: How do rolling updates and rollbacks work?

**Answer:**

**Rolling updates** let you update a Deployment's Pod template (e.g., new image version) with zero downtime. The Deployment controller replaces Pods gradually rather than all at once.

**How it works step-by-step:**

1. You change the Pod template (e.g., `kubectl set image deployment/myapp app=myapp:2.0`).
2. The Deployment controller creates a **new ReplicaSet** with the updated template.
3. It scales up the new ReplicaSet and scales down the old one incrementally.
4. At each step, it respects:
   - `maxSurge` -- Maximum number of extra Pods above the desired count (default 25%).
   - `maxUnavailable` -- Maximum number of Pods that can be unavailable (default 25%).
5. Once all new Pods are ready and all old Pods are terminated, the rollout is complete.

**Example with zero downtime:**

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # allow 1 extra Pod during update (4 total)
      maxUnavailable: 0    # never go below 3 ready Pods
```

**Rollbacks:**

Kubernetes keeps previous ReplicaSets (controlled by `revisionHistoryLimit`, default 10). Each ReplicaSet represents a revision. A rollback simply scales a previous ReplicaSet back up and scales the current one down.

```bash
# Check rollout status
kubectl rollout status deployment myapp

# View revision history
kubectl rollout history deployment myapp

# See details of a specific revision
kubectl rollout history deployment myapp --revision=2

# Rollback to the previous revision
kubectl rollout undo deployment myapp

# Rollback to a specific revision
kubectl rollout undo deployment myapp --to-revision=2

# Pause and resume a rollout (for canary-style testing)
kubectl rollout pause deployment myapp
kubectl rollout resume deployment myapp
```

Under the hood, `kubectl rollout undo` does not revert the manifest. It updates the Deployment's Pod template to match the old ReplicaSet's template, which triggers a new rollout that reuses the old ReplicaSet.

---

### Q20: How do command and args work in a Pod spec?

**Answer:**

In a Pod spec, `command` and `args` override the container image's `ENTRYPOINT` and `CMD`.

**Mapping to Dockerfile:**

| Dockerfile | Pod spec | Purpose |
|------------|----------|---------|
| `ENTRYPOINT` | `command` | The executable to run |
| `CMD` | `args` | Default arguments passed to the executable |

**Override rules:**

| `command` set? | `args` set? | What runs |
|----------------|-------------|-----------|
| No | No | Image's `ENTRYPOINT` + `CMD` |
| Yes | No | Pod's `command` only (image CMD ignored) |
| No | Yes | Image's `ENTRYPOINT` + Pod's `args` |
| Yes | Yes | Pod's `command` + Pod's `args` |

**Examples:**

```yaml
# Use the image defaults (ENTRYPOINT + CMD)
containers:
  - name: app
    image: nginx:1.25

# Override just the arguments
containers:
  - name: app
    image: myapp:1.0
    args: ["--port", "9090", "--verbose"]

# Override the entire command and arguments
containers:
  - name: debug
    image: busybox
    command: ["sh", "-c"]
    args: ["echo hello && sleep 3600"]

# Run a one-off task
containers:
  - name: migrate
    image: myapp:1.0
    command: ["python", "manage.py", "migrate"]
```

Key detail: `command` and `args` are arrays. Each element is a separate argument. Do not try to pass a full command string as a single element unless you prefix it with a shell (`sh -c`).

---

### Q21: What are all the ways to inject environment variables into a Pod?

**Answer:**

There are five ways to set environment variables in a container spec.

**1. Static key-value pairs (`env`)**

Hardcoded directly in the manifest:

```yaml
env:
  - name: APP_ENV
    value: "production"
  - name: LOG_LEVEL
    value: "info"
```

**2. From a ConfigMap (`configMapKeyRef` / `configMapRef`)**

Single key:
```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
```

All keys at once:
```yaml
envFrom:
  - configMapRef:
      name: app-config
```

Each key in the ConfigMap becomes an environment variable.

**3. From a Secret (`secretKeyRef` / `secretRef`)**

Single key:
```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: password
```

All keys at once:
```yaml
envFrom:
  - secretRef:
      name: db-creds
```

**4. From Pod fields -- Downward API (`fieldRef`)**

Exposes Pod metadata as environment variables:

```yaml
env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
  - name: SERVICE_ACCOUNT
    valueFrom:
      fieldRef:
        fieldPath: spec.serviceAccountName
```

**5. From container resource limits (`resourceFieldRef`)**

Exposes the container's own resource requests and limits:

```yaml
env:
  - name: CPU_LIMIT
    valueFrom:
      resourceFieldRef:
        containerName: app
        resource: limits.cpu
  - name: MEMORY_LIMIT
    valueFrom:
      resourceFieldRef:
        containerName: app
        resource: limits.memory
```

**Summary table:**

| Method | Source | Use case |
|--------|--------|----------|
| `value` | Inline string | Simple, static values |
| `configMapKeyRef` / `configMapRef` | ConfigMap | Non-sensitive config shared across Pods |
| `secretKeyRef` / `secretRef` | Secret | Passwords, tokens, certificates |
| `fieldRef` | Pod metadata (Downward API) | Pod name, namespace, IP, node name |
| `resourceFieldRef` | Container resources | Expose CPU/memory limits to the app |

`envFrom` injects all keys from a ConfigMap or Secret. You can add `prefix` to namespace them:

```yaml
envFrom:
  - configMapRef:
      name: app-config
    prefix: APP_
```

This turns a ConfigMap key `PORT` into the variable `APP_PORT`.

---

### Q22: What are init containers and how do they work?

**Answer:**

Init containers are specialized containers that run **before** the main application containers start. They run to completion, one at a time, in the order they are defined. The main containers only start after all init containers have succeeded.

**Key properties:**
- Run sequentially (not in parallel)
- Must exit with code 0 to be considered successful
- If an init container fails, the kubelet retries it according to the Pod's `restartPolicy`
- Can use a different image from the main container
- Have access to the same volumes as the main containers
- Do not support readiness, liveness, or startup probes (they are not long-running)

**Common use cases:**
- **Wait for dependencies** -- Block startup until a database or API is reachable
- **Run database migrations** -- Apply schema changes before the app starts
- **Populate shared volumes** -- Download config files or clone a Git repo
- **Set permissions** -- Fix file ownership or permissions on mounted volumes

**Example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ["sh", "-c"]
      args:
        - |
          until nc -z postgres-svc 5432; do
            echo "waiting for database..."
            sleep 2
          done
    - name: run-migrations
      image: myapp:1.0
      command: ["python", "manage.py", "migrate"]
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8080
```

In this example, `wait-for-db` runs first until the database is reachable, then `run-migrations` applies schema changes, and only then does the `app` container start.

**Init containers vs sidecar containers:**

| Aspect | Init container | Sidecar container |
|--------|---------------|-------------------|
| When it runs | Before main containers | Alongside main containers |
| Lifecycle | Runs to completion, then exits | Runs for the lifetime of the Pod |
| Use case | Setup and preconditions | Ongoing support (logging, proxying) |

---

### Q23: What is a sidecar container?

**Answer:**

A sidecar container runs **alongside** the main application container in the same Pod for the entire Pod's lifetime. It extends or enhances the application without modifying its code.

Because containers in a Pod share the same network namespace and volumes, the sidecar can:
- Access the same `localhost` network
- Read/write to shared volumes
- Start and stop with the same lifecycle

**Common sidecar patterns:**

| Pattern | Sidecar does | Example |
|---------|-------------|---------|
| **Proxy / Service mesh** | Handles traffic routing, mTLS, retries | Envoy (Istio), Linkerd proxy |
| **Log shipping** | Reads log files from a shared volume and forwards them | Fluentd, Filebeat |
| **Monitoring agent** | Collects metrics and sends to a backend | Prometheus exporter |
| **Config reload** | Watches for config changes and signals the main app | Config watcher |

**Example -- Envoy proxy sidecar:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8080
    - name: envoy
      image: envoyproxy/envoy:v1.28
      ports:
        - containerPort: 9901
      volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
  volumes:
    - name: envoy-config
      configMap:
        name: envoy-config
```

**Native sidecar containers (Kubernetes 1.28+):**

Kubernetes introduced a `restartPolicy: Always` field on init containers, making them run for the Pod's lifetime instead of exiting. This gives sidecars proper lifecycle ordering -- they start before the main container and stop after it.

```yaml
initContainers:
  - name: log-agent
    image: fluentd:latest
    restartPolicy: Always   # makes this a native sidecar
```

This solves the old problem of sidecar containers exiting before the main container finishes (or the main container starting before the sidecar is ready).

---

### Q24: What is autoscaling in Kubernetes?

**Answer:**

Kubernetes supports three levels of autoscaling:

**1. Horizontal Pod Autoscaler (HPA)**

Scales the number of Pod replicas based on observed metrics (CPU, memory, or custom metrics).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

This scales the Deployment between 2 and 10 replicas, targeting 70% average CPU utilization. HPA requires the Metrics Server to be installed.

```bash
# Create an HPA imperatively
kubectl autoscale deployment myapp --min=2 --max=10 --cpu-percent=70

# Check HPA status
kubectl get hpa
```

Do not set `spec.replicas` in the Deployment when using HPA -- it will conflict with the autoscaler.

**2. Vertical Pod Autoscaler (VPA)**

Adjusts the CPU and memory **requests and limits** of containers based on actual usage. It does not change the replica count.

- Analyzes historical resource usage
- Recommends or automatically applies new resource values
- Requires Pod restarts to apply changes (it evicts and recreates Pods)

VPA is useful when you are unsure what resource requests to set. It is not built into Kubernetes by default and must be installed separately.

Do not use HPA and VPA together on the same metric (e.g., both scaling on CPU).

**3. Cluster Autoscaler**

Scales the number of **nodes** in the cluster.

- When Pods cannot be scheduled because no node has enough resources, it provisions a new node from the cloud provider.
- When nodes are underutilized and their Pods can be moved elsewhere, it removes the node.
- Works with cloud provider APIs (AWS ASG, GCP MIG, Azure VMSS).

**How the three work together:**

```
Traffic increases
      |
HPA: adds more Pod replicas
      |
No node has room for the new Pods
      |
Cluster Autoscaler: provisions a new node
      |
Scheduler: places Pods on the new node
```

**Comparison:**

| Autoscaler | What it scales | Based on | Built-in |
|------------|---------------|----------|----------|
| HPA | Pod replicas | CPU, memory, custom metrics | Yes |
| VPA | Container resources (requests/limits) | Historical usage | No (add-on) |
| Cluster Autoscaler | Nodes | Pending Pods / node utilization | No (add-on) |

---

## Storage

### Q25: What are the different volume types in Kubernetes?

**Answer:**

Kubernetes supports many volume types. The most important ones for interviews:

**Ephemeral volumes (deleted with the Pod):**

| Type | Description |
|------|-------------|
| `emptyDir` | Empty directory created when the Pod starts. Shared between containers in the same Pod. Deleted when the Pod is removed. |
| `emptyDir` with `medium: Memory` | Same as above but backed by tmpfs (RAM). Fast but counts against the container's memory limit. |

```yaml
volumes:
  - name: cache
    emptyDir: {}
  - name: scratch
    emptyDir:
      medium: Memory
      sizeLimit: 100Mi
```

**Persistent volumes (survive Pod deletion):**

| Type | Description |
|------|-------------|
| `persistentVolumeClaim` | References a PVC, which binds to a PV. The standard way to use persistent storage. |
| `hostPath` | Mounts a file or directory from the host node's filesystem. Dangerous in production -- ties the Pod to a specific node. |
| `nfs` | Mounts an NFS share. Supports `ReadWriteMany` access mode. |

**Configuration volumes (read-only data injected into the Pod):**

| Type | Description |
|------|-------------|
| `configMap` | Mounts ConfigMap keys as files. |
| `secret` | Mounts Secret keys as files on a tmpfs (never written to disk). |
| `downwardAPI` | Exposes Pod metadata (labels, annotations, resource limits) as files. |
| `projected` | Combines multiple sources (ConfigMap, Secret, downwardAPI, ServiceAccountToken) into a single mount. |

```yaml
volumes:
  - name: config
    projected:
      sources:
        - configMap:
            name: app-config
        - secret:
            name: app-secret
```

**Cloud provider volumes (via CSI):**

| Provider | CSI driver |
|----------|-----------|
| AWS | `ebs.csi.aws.com` (EBS), `efs.csi.aws.com` (EFS) |
| GCP | `pd.csi.storage.gke.io` (Persistent Disk) |
| Azure | `disk.csi.azure.com` (Azure Disk), `file.csi.azure.com` (Azure Files) |

These are used through PVCs and StorageClasses, not referenced directly.

---

### Q26: What are PersistentVolumes, PersistentVolumeClaims, and how do they work together?

**Answer:**

The PV/PVC model separates storage provisioning from storage consumption.

- **PersistentVolume (PV)** -- A piece of storage in the cluster, provisioned by an admin or dynamically by a StorageClass. It is a cluster-scoped resource.
- **PersistentVolumeClaim (PVC)** -- A request for storage by a user. It is namespace-scoped. Pods reference PVCs, not PVs directly.

**Lifecycle:**

```
Admin creates PV (or StorageClass provisions one)
      |
User creates PVC with size and access mode
      |
Control plane binds PVC to a matching PV
      |
Pod mounts the PVC as a volume
      |
Pod is deleted --> PVC remains --> data persists
      |
PVC is deleted --> reclaim policy decides PV fate
```

**PV example (static provisioning):**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/my-pv
```

**PVC example:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**Mounting in a Pod:**

```yaml
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /app/data
```

**Access modes:**

| Mode | Abbreviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | RWO | Mounted read-write by a single node |
| ReadOnlyMany | ROX | Mounted read-only by many nodes |
| ReadWriteMany | RWX | Mounted read-write by many nodes (NFS, EFS) |
| ReadWriteOncePod | RWOP | Mounted read-write by a single Pod (k8s 1.27+) |

**Reclaim policies:**

| Policy | What happens when PVC is deleted |
|--------|--------------------------------|
| Retain | PV remains with data intact. Admin must manually clean up. |
| Delete | PV and underlying storage are deleted. Default for dynamic provisioning. |
| Recycle | Deprecated. Was a basic `rm -rf /volume/*`. |

---

### Q27: What are StorageClasses and dynamic provisioning?

**Answer:**

A StorageClass defines how storage is dynamically provisioned when a PVC is created. Instead of an admin manually creating PVs, the StorageClass tells Kubernetes which provisioner to use and with what parameters.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "5000"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**Key fields:**

| Field | Purpose |
|-------|---------|
| `provisioner` | The CSI driver or plugin that creates the actual storage |
| `parameters` | Provider-specific settings (disk type, IOPS, etc.) |
| `reclaimPolicy` | What happens to the PV when the PVC is deleted |
| `volumeBindingMode` | `Immediate` (bind PV right away) or `WaitForFirstConsumer` (bind when a Pod uses it -- avoids zone mismatch) |
| `allowVolumeExpansion` | Whether PVCs using this class can be resized |

**PVC referencing a StorageClass:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

When this PVC is created, the StorageClass automatically provisions a 20Gi gp3 EBS volume.

**Default StorageClass:** If a PVC does not specify `storageClassName`, the cluster's default StorageClass is used (the one annotated with `storageclass.kubernetes.io/is-default-class: "true"`).

---

### Q28: What is a StatefulSet and how does it handle storage?

**Answer:**

A StatefulSet manages stateful applications that need stable identities and persistent storage.

**How it differs from a Deployment:**

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random suffix (`myapp-7b4d5-xk2lp`) | Ordered index (`myapp-0`, `myapp-1`, `myapp-2`) |
| Startup order | All Pods start in parallel | Pods start sequentially (0, then 1, then 2) |
| Stable network ID | No | Yes, via a headless Service (`myapp-0.myapp-svc`) |
| Storage | Shared PVC or no PVC | Each Pod gets its own PVC via `volumeClaimTemplates` |
| Deletion order | Random | Reverse order (2, then 1, then 0) |

**volumeClaimTemplates:**

Each Pod gets a unique PVC that follows it across rescheduling:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-svc
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 10Gi
```

This creates PVCs named `data-postgres-0`, `data-postgres-1`, `data-postgres-2`. If `postgres-1` is rescheduled to another node, it reattaches to `data-postgres-1`.

**Use cases:** Databases (PostgreSQL, MySQL), message brokers (Kafka), distributed stores (etcd, Elasticsearch).

Deleting a StatefulSet does **not** delete its PVCs. This is by design to prevent accidental data loss.

---

## Networking

### Q29: Explain the Kubernetes networking model.

**Answer:**

Kubernetes imposes three fundamental rules:

1. **Every Pod gets its own IP address.**
2. **All Pods can communicate with all other Pods without NAT** (across any node).
3. **The IP a Pod sees for itself is the same IP other Pods see for it.**

**How traffic flows at each level:**

**Pod-to-Pod on the same node:**
- Containers within a Pod communicate via `localhost`.
- Pods on the same node communicate through a virtual bridge (e.g., `cbr0`). The CNI plugin sets this up.

**Pod-to-Pod across nodes:**
- The CNI plugin creates an overlay network (VXLAN, IPIP, WireGuard) or uses native routing (BGP with Calico).
- Each node gets a Pod CIDR (e.g., `10.244.1.0/24`), and the CNI plugin ensures routes exist between them.

**Pod-to-Service:**
- Pods access Services via ClusterIP or DNS name.
- `kube-proxy` uses iptables/IPVS rules to NAT the Service IP to a backing Pod IP.

**External-to-Pod:**
- NodePort: External traffic hits `<NodeIP>:<NodePort>`, kube-proxy forwards to a Pod.
- LoadBalancer: Cloud LB forwards to NodePorts.
- Ingress: An Ingress controller (nginx, Traefik) terminates HTTP/HTTPS and routes to Services.

**Key IP ranges:**

| Range | Purpose | Example |
|-------|---------|---------|
| Pod CIDR | IP addresses for Pods | `10.244.0.0/16` |
| Service CIDR | ClusterIP addresses for Services | `10.96.0.0/12` |
| Node IPs | Actual node addresses | `192.168.1.0/24` |

---

### Q30: What are NetworkPolicies?

**Answer:**

A NetworkPolicy controls traffic flow to and from Pods at the IP/port level. By default, all Pods can communicate with all other Pods. NetworkPolicies add firewall-like rules.

**Key concepts:**
- NetworkPolicies are namespace-scoped.
- They select Pods using `podSelector`.
- They require a CNI plugin that supports them (Calico, Cilium, Weave). Flannel does **not** support NetworkPolicies.
- If any NetworkPolicy selects a Pod, all traffic **not explicitly allowed** is denied (default deny for the selected direction).

**Example -- deny all ingress to a namespace:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}   # selects all Pods in the namespace
  policyTypes:
    - Ingress
```

**Example -- allow traffic only from frontend Pods to backend Pods on port 8080:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

**Example -- allow DNS egress (required if you deny all egress):**

```yaml
egress:
  - to: []
    ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

**Common patterns:**

| Pattern | podSelector | policyTypes | Rules |
|---------|------------|-------------|-------|
| Deny all ingress | `{}` | Ingress | No `ingress` rules |
| Deny all egress | `{}` | Egress | No `egress` rules |
| Allow from specific Pods | `app: backend` | Ingress | `from: podSelector: app: frontend` |
| Allow from a namespace | `app: backend` | Ingress | `from: namespaceSelector: name: monitoring` |
| Allow to external CIDR | `app: backend` | Egress | `to: ipBlock: cidr: 10.0.0.0/8` |

---

### Q31: What is Ingress and how does it differ from a Service?

**Answer:**

An Ingress is an API object that manages external HTTP/HTTPS access to Services. It provides capabilities that Services alone do not have.

**What Ingress adds over a plain Service:**

| Feature | Service (NodePort / LB) | Ingress |
|---------|------------------------|---------|
| Layer | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| Host-based routing | No | Yes (`app.example.com` vs `api.example.com`) |
| Path-based routing | No | Yes (`/app` vs `/api`) |
| TLS termination | No (needs external LB) | Yes (terminates HTTPS at the Ingress controller) |
| Single entry point | One LB per Service | One LB for many Services |

**Ingress requires an Ingress controller** (a Pod that reads Ingress resources and configures routing). Common controllers:

| Controller | Notes |
|-----------|-------|
| NGINX Ingress | Most widely used, community and NGINX Inc versions |
| Traefik | Auto-discovery, built-in Let's Encrypt |
| HAProxy | High performance |
| AWS ALB Ingress | Provisions AWS ALBs directly |

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

**Path types:**

| Type | Behavior |
|------|----------|
| `Exact` | Matches the path exactly (e.g., `/api` but not `/api/v1`) |
| `Prefix` | Matches the path prefix (e.g., `/api` matches `/api`, `/api/v1`, `/api/users`) |
| `ImplementationSpecific` | Behavior depends on the Ingress controller |

---

### Q32: How does DNS work inside a Kubernetes cluster?

**Answer:**

Kubernetes runs **CoreDNS** as a Deployment in `kube-system`. Every Pod is configured to use CoreDNS as its DNS server (via `/etc/resolv.conf`).

**What CoreDNS resolves:**

| Record | Format | Example |
|--------|--------|---------|
| Service | `<service>.<namespace>.svc.cluster.local` | `myapp.production.svc.cluster.local` |
| Pod | `<pod-ip-dashed>.<namespace>.pod.cluster.local` | `10-244-1-5.production.pod.cluster.local` |
| StatefulSet Pod | `<pod-name>.<headless-service>.<namespace>.svc.cluster.local` | `postgres-0.postgres-svc.production.svc.cluster.local` |
| Headless Service | Returns Pod IPs directly (A records for each Pod) | `postgres-svc.production.svc.cluster.local` |
| ExternalName Service | Returns a CNAME record | `mydb.production.svc.cluster.local` -> `db.example.com` |

**DNS search domains:**

Pods can use short names because `/etc/resolv.conf` includes search domains:

```
search production.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
```

This means:
- `myapp` resolves by trying `myapp.production.svc.cluster.local` first
- `myapp.other-ns` resolves as `myapp.other-ns.svc.cluster.local`

**dnsPolicy options:**

| Policy | Behavior |
|--------|----------|
| `ClusterFirst` (default) | Use CoreDNS for cluster names, forward external names to upstream |
| `Default` | Use the node's DNS config (`/etc/resolv.conf` on the node) |
| `None` | No auto-config. You must provide `dnsConfig` manually |
| `ClusterFirstWithHostNet` | Like `ClusterFirst` but for Pods using `hostNetwork: true` |

**Debugging DNS:**

```bash
kubectl run debug --rm -it --image=busybox -- nslookup myapp.production.svc.cluster.local

# Check CoreDNS Pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---
