# Kubernetes -- Scenario-Based Questions

Real-world troubleshooting and problem-solving exercises.

---

### Scenario 1: Pod stuck in CrashLoopBackOff

**Situation:** A Deployment's Pods are in `CrashLoopBackOff`. The application worked fine locally.

**How would you troubleshoot this?**

**Answer:**

1. Check the Pod events:
   ```bash
   kubectl describe pod <pod-name>
   ```

2. Check container logs:
   ```bash
   kubectl logs <pod-name> --previous
   ```

3. Common causes:
   - Missing environment variables or ConfigMap/Secret references
   - Incorrect command/entrypoint in the container image
   - Application crash due to missing dependencies or wrong config
   - Insufficient memory causing OOMKill (check `Last State: OOMKilled`)

4. If OOMKilled, increase memory limits in the Pod spec:
   ```yaml
   resources:
     limits:
       memory: "512Mi"
   ```

---

### Scenario 2: Service not reachable

**Situation:** You deployed an application and a Service, but you cannot reach the application from within the cluster.

**How would you debug this?**

**Answer:**

1. Verify the Service exists and has endpoints:
   ```bash
   kubectl get svc <service-name>
   kubectl get endpoints <service-name>
   ```

2. If endpoints are empty, the Service selector does not match the Pod labels. Compare:
   ```bash
   kubectl get svc <service-name> -o yaml | grep -A5 selector
   kubectl get pods --show-labels
   ```

3. Test connectivity from another Pod:
   ```bash
   kubectl run debug --rm -it --image=busybox -- wget -qO- http://<service-name>:<port>
   ```

4. Check if the container is listening on the correct port:
   ```bash
   kubectl exec <pod-name> -- ss -tlnp
   ```

---
