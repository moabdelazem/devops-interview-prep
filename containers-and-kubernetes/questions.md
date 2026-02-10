# Containers & Kubernetes -- Interview Questions

General and conceptual interview questions with clear answers.

---

## Containers

### Q1: What is a container and how does it differ from a virtual machine?

**Answer:**
A container is a lightweight, isolated process that shares the host OS kernel. It uses Linux kernel features -- namespaces for isolation and cgroups for resource limits -- to run applications in their own environment without a full guest OS.

Key differences from VMs:

| Aspect | Container | Virtual Machine |
|--------|-----------|-----------------|
| Isolation | Process-level (shared kernel) | Hardware-level (hypervisor) |
| Startup time | Seconds | Minutes |
| Size | MBs (image layers) | GBs (full OS disk) |
| Overhead | Minimal | Higher (guest OS) |

---

### Q2: What are namespaces and cgroups?

**Answer:**

- **Namespaces** isolate what a process can see: PID, network, mount, user, UTS, and IPC namespaces each limit visibility to a subset of system resources.
- **Cgroups (control groups)** limit how much a process can use: CPU, memory, disk I/O, and network bandwidth.

Together, they form the foundation of container isolation on Linux.

---

### Q3: What is the difference between a Docker image and a container?

**Answer:**

- An **image** is a read-only template made of layered filesystems. It contains the application code, runtime, libraries, and dependencies.
- A **container** is a running instance of an image. It adds a writable layer on top of the image layers.

You can create many containers from the same image.

---

### Q4: What are the best practices for writing a Dockerfile?

**Answer:**

**1. Use multi-stage builds** -- Separate build-time dependencies from the final runtime image. This keeps the image small and free of compilers, build tools, etc.

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
COPY --from=builder /app/myapp /usr/local/bin/
CMD ["myapp"]
```

**2. Use minimal base images** -- Prefer `ubi-minimal`, `alpine`, or `distroless` over full OS images. Smaller images mean faster pulls and a smaller attack surface.

**3. Order layers for cache efficiency** -- Place instructions that change less frequently (installing OS packages) before instructions that change often (copying source code).

```dockerfile
# Good: dependencies cached separately from source
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o myapp
```

**4. Combine RUN instructions** -- Each `RUN` creates a new layer. Combine related commands and clean up in the same layer.

```dockerfile
RUN dnf install -y curl && \
    dnf clean all && \
    rm -rf /var/cache/dnf
```

**5. Use .dockerignore** -- Exclude files that should not be in the build context (`.git`, test files, docs, local configs).

```
.git
*.md
tests/
.env
```

**6. Do not run as root** -- Create and switch to a non-root user.

```dockerfile
RUN useradd -r -s /bin/false appuser
USER appuser
```

**7. Set explicit tags, never use `latest`** -- Pin base image versions to ensure reproducible builds.

```dockerfile
FROM node:20.11-alpine3.19
```

**8. Use COPY instead of ADD** -- `COPY` is straightforward. `ADD` has implicit behavior (auto-extracting archives, fetching URLs) that can cause surprises.

**9. Add health checks** -- Let the orchestrator know if the container is healthy.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s \
  CMD curl -f http://localhost:8080/health || exit 1
```

**10. Use labels for metadata** -- Makes images easier to manage and audit.

```dockerfile
LABEL maintainer="you@example.com"
LABEL version="1.0.0"
```

---

### Q5: What is the difference between ENTRYPOINT and CMD?

**Answer:**

- **CMD** sets the default command and arguments. It can be overridden entirely at `docker run`.
- **ENTRYPOINT** sets the main executable. Arguments passed at `docker run` are appended to it.

| Instruction | Override behavior |
|-------------|-------------------|
| `CMD ["nginx"]` | `docker run myimage bash` runs `bash` instead |
| `ENTRYPOINT ["nginx"]` | `docker run myimage -g "daemon off;"` runs `nginx -g "daemon off;"` |

Best practice: use `ENTRYPOINT` for the main process and `CMD` for default flags that users can override.

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

---

### Q6: What is a Docker layer and how does caching work?

**Answer:**

Each instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, etc.) creates a read-only layer. Docker caches layers and reuses them if the instruction and its context have not changed.

Cache is invalidated when:

- The instruction text changes
- Files referenced by `COPY` / `ADD` change (checksum comparison)
- Any parent layer is invalidated (cache bust cascades down)

This is why layer ordering matters -- put rarely changing layers first (OS packages) and frequently changing layers last (application code).

---

### Q7: What is the difference between Docker volumes, bind mounts, and tmpfs?

**Answer:**

| Type | Where data lives | Managed by Docker | Persists after container removal |
|------|------------------|-------------------|----------------------------------|
| Volume | Docker-managed area (`/var/lib/docker/volumes/`) | Yes | Yes |
| Bind mount | Any path on the host filesystem | No | Yes (on host) |
| tmpfs | In-memory only | No | No |

When to use each:

- **Volumes** -- Default choice for persistent data. Portable, easy to back up, works across Linux and macOS/Windows.
  ```bash
  docker run -v mydata:/app/data myimage
  ```

- **Bind mounts** -- When you need the container to access a specific host directory (e.g., mounting source code during development).
  ```bash
  docker run -v /home/user/project:/app myimage
  ```

- **tmpfs** -- For sensitive or throwaway data that should never be written to disk (e.g., secrets, session tokens).
  ```bash
  docker run --tmpfs /app/tmp myimage
  ```

---

### Q8: What is container image scanning and why does it matter?

**Answer:**

Image scanning analyzes container images for known vulnerabilities (CVEs) in OS packages and application dependencies. It is a critical part of a secure CI/CD pipeline.

**Common tools:**

| Tool | Description |
|------|-------------|
| Trivy | Open-source, scans images, filesystems, and IaC. Fast and easy to integrate. |
| Grype | Open-source by Anchore. Scans images and SBOMs. |
| Clair | Open-source, designed for integration with container registries. |

**Example with Trivy:**
```bash
# Scan a local image
trivy image myapp:latest

# Scan and fail the build if critical/high CVEs are found
trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:latest
```

**Best practices:**
- Scan images in CI before pushing to a registry.
- Use minimal base images to reduce the number of packages that can have CVEs.
- Rebuild and re-scan images regularly, even if your code has not changed, to catch newly disclosed CVEs.

---

### Q9: What are rootless containers and why do they matter?

**Answer:**

Rootless containers run the entire container runtime (and the containers themselves) without requiring root privileges on the host.

**Why it matters:**
- In traditional Docker, the daemon runs as root. If an attacker escapes the container, they have root access to the host.
- Rootless mode eliminates this risk by using user namespaces -- the "root" inside the container maps to an unprivileged user on the host.

**Docker rootless mode:**
```bash
# Install rootless Docker
dockerd-rootless-setuptool.sh install

# Containers run under your user, no sudo needed
docker run -d nginx
```

**Podman is rootless by default** -- It has no daemon and runs containers under the calling user without any extra setup.

```bash
# No root, no daemon
podman run -d nginx
```

**Trade-offs:**
- Some networking features (binding to ports below 1024) require workarounds.
- Filesystem performance may be slightly lower due to user namespace UID mapping (using `fuse-overlayfs`).
- On RHEL, rootless Podman is fully supported and recommended.

---

## Kubernetes

### Q10: What is Kubernetes and why is it needed?

**Answer:**
Kubernetes (k8s) is an open-source container orchestration platform. It automates deployment, scaling, and management of containerized applications.

It is needed because running containers manually does not scale. Kubernetes handles:

- Service discovery and load balancing
- Automated rollouts and rollbacks
- Self-healing (restarting failed containers)
- Horizontal scaling
- Secret and configuration management

---

### Q11: Explain the Kubernetes architecture.

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

### Q12: What is a Pod?

**Answer:**
A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share:

- The same network namespace (same IP, can communicate via localhost)
- The same storage volumes
- The same lifecycle

Most workloads run a single container per Pod, but sidecar patterns (logging, proxying) use multi-container Pods.

---
