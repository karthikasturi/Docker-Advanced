### ✅ **Docker Swarm – Demo & Lab Guide**

#### **Lab 1: Swarm Cluster Setup**

**Objective:** Set up a 3-node Swarm cluster with 1 manager and 2 workers.

```bash
# Initialize Swarm on Manager Node
docker swarm init --advertise-addr <manager_ip>

# Get join token for workers
docker swarm join-token worker

# Join on Worker Nodes (copy from above)
docker swarm join --token <token> <manager_ip>:2377

# Verify
docker node ls
```

---

#### **Lab 2: Deploy Docker Service**

```bash
# Create a service
docker service create --name web --replicas 3 -p 80:80 nginx

# View service and tasks
docker service ls
docker service ps web

# Scale service
docker service scale web=5

# Rolling update (e.g. new version)
docker service update --image nginx:1.25-alpine web
```

---

#### **Lab 3: Deploy Stack with Compose**

**docker-compose.yml** example:

```yaml
version: '3.8'

services:
  web:
    image: nginx
    ports:
      - "80:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 2
        delay: 10s

  redis:
    image: redis:alpine
    deploy:
      replicas: 1
```

```bash
# Deploy stack 
docker stack deploy -c docker-compose.yml mystack 
# Monitor 
docker stack ls
docker stack services mystack 
docker stack ps mystack
```

---

#### **Lab 4: Simulate Node Failure**

```bash
# Stop Docker on a worker 
sudo systemctl stop docker 
# Monitor service redistribution 
docker node ls 
docker service ps web
```
