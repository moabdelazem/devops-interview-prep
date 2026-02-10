# Containers -- Scenario-Based Questions

Real-world troubleshooting and problem-solving exercises.

---

### Scenario 1: Docker image too large

**Situation:** Your Docker image is 1.5 GB and takes too long to pull in production.

**How would you reduce its size?**

**Answer:**

1. Use a multi-stage build to separate build dependencies from runtime:
   ```dockerfile
   FROM golang:1.21 AS builder
   WORKDIR /app
   COPY . .
   RUN go build -o myapp

   FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
   COPY --from=builder /app/myapp /usr/local/bin/
   CMD ["myapp"]
   ```

2. Use a minimal base image (e.g., `ubi-minimal`, `alpine`, `distroless`).

3. Combine `RUN` instructions to reduce layers:
   ```dockerfile
   RUN dnf install -y curl && dnf clean all
   ```

4. Add a `.dockerignore` to exclude build artifacts, `.git`, and test files.

---

### Scenario 2: Container running but application not responding

**Situation:** Your container is in `running` state but the application inside is not responding to HTTP requests.

**How would you debug this?**

**Answer:**

1. Check the container logs for errors:
   ```bash
   docker logs <container>
   ```

2. Verify the process is actually running inside:
   ```bash
   docker exec <container> ps aux
   ```

3. Check if the application is listening on the expected port:
   ```bash
   docker exec <container> ss -tlnp
   ```

4. Verify port mapping is correct:
   ```bash
   docker port <container>
   ```

5. Common causes:
   - Application is binding to `127.0.0.1` instead of `0.0.0.0` (unreachable from outside the container)
   - Wrong port mapped in `docker run -p`
   - Application started but hit a deadlock or is waiting on a missing dependency (database, config server)

---

### Scenario 3: Container port conflict

**Situation:** You try to start a container with `-p 8080:80` but get `bind: address already in use`.

**How would you resolve this?**

**Answer:**

1. Find what is using port 8080 on the host:
   ```bash
   ss -tlnp | grep 8080
   ```

2. If it is another container:
   ```bash
   docker ps --format '{{.Names}} {{.Ports}}' | grep 8080
   ```

3. Options:
   - Stop the conflicting container or process
   - Map to a different host port: `-p 9090:80`
   - Let Docker pick a random available port: `-p 80` (without host port)

---

### Scenario 4: Container data disappears after restart

**Situation:** A database container loses all its data every time it is restarted.

**Why is this happening and how do you fix it?**

**Answer:**

Container filesystems are ephemeral. When a container is removed, its writable layer is deleted.

Fix by using a named volume:

```bash
# Bad: data lives in the container's writable layer
docker run -d --name db postgres:15

# Good: data persists in a Docker volume
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:15
```

Even with `docker stop` and `docker start`, the writable layer survives. But if the container is removed (`docker rm`), the data is gone unless a volume is used.

---
