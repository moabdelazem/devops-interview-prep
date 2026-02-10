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

### Q3: What is a Pod?

**Answer:**
A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share:

- The same network namespace (same IP, can communicate via localhost)
- The same storage volumes
- The same lifecycle

Most workloads run a single container per Pod, but sidecar patterns (logging, proxying) use multi-container Pods.

---
