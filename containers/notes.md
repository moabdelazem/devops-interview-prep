# Containers -- Notes & Cheat Sheet

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
