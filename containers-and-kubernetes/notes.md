# Containers & Kubernetes -- Notes & Cheat Sheet

Quick-reference commands and concepts for revision.

---

## Docker / Podman CLI

```bash
# Build an image
docker build -t myapp:latest .

# Run a container
docker run -d --name myapp -p 8080:80 myapp:latest

# List running containers
docker ps

# View logs
docker logs -f <container>

# Execute a command inside a running container
docker exec -it <container> /bin/bash

# Remove all stopped containers
docker container prune

# List images
docker images

# Remove an image
docker rmi <image>
```

---

## Kubernetes -- kubectl Essentials

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes

# Working with Pods
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod> -c <container>
kubectl exec -it <pod> -- /bin/sh

# Deployments
kubectl create deployment myapp --image=myapp:latest
kubectl scale deployment myapp --replicas=3
kubectl rollout status deployment myapp
kubectl rollout undo deployment myapp

# Services
kubectl expose deployment myapp --port=80 --target-port=8080 --type=ClusterIP
kubectl get svc

# ConfigMaps and Secrets
kubectl create configmap myconfig --from-literal=key=value
kubectl create secret generic mysecret --from-literal=password=s3cret

# Debugging
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl top nodes

# Apply and delete manifests
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml

# Context and namespace
kubectl config get-contexts
kubectl config use-context <context>
kubectl get pods -n <namespace>
```

---

## Key Kubernetes Objects

| Object | Purpose |
|--------|---------|
| Pod | Smallest unit; runs one or more containers |
| ReplicaSet | Ensures a specified number of Pod replicas |
| Deployment | Manages ReplicaSets; handles rollouts and rollbacks |
| Service | Stable network endpoint for a set of Pods |
| Ingress | HTTP/HTTPS routing to Services |
| ConfigMap | Non-sensitive configuration as key-value pairs |
| Secret | Sensitive data (base64-encoded, not encrypted by default) |
| PersistentVolume | Cluster-level storage resource |
| PersistentVolumeClaim | Request for storage by a Pod |
| HPA | Horizontal Pod Autoscaler; scales based on CPU/memory |
| Namespace | Logical isolation within a cluster |

---

## Service Types

| Type | Description |
|------|-------------|
| ClusterIP | Internal-only; default type |
| NodePort | Exposes on each node's IP at a static port (30000-32767) |
| LoadBalancer | Provisions an external load balancer (cloud provider) |
| ExternalName | Maps to a DNS name (CNAME record) |

---
