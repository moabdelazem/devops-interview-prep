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
