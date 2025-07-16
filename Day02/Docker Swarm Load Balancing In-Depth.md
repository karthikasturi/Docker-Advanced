# Docker Swarm Load Balancing – In-Depth

## 1. VIP-Based Load Balancing (Virtual IP)

Docker Swarm assigns a Virtual IP (VIP) to each service, enabling internal service discovery and round-robin load balancing across service tasks using the Linux IPVS system.

**How it works:**

- Containers resolve service names to VIPs.

- Traffic to a VIP is distributed using IPVS.

- Default algorithm: Round Robin (not configurable via Docker CLI).

**Pros:**

- Simple and automatic.

- No external configuration needed.

- DNS-based service discovery.

**Cons:**

- Limited to internal Swarm communication.

- No algorithm customization.

- Harder to debug.

---

## 2. Routing Mesh (for Published Ports)

Swarm's routing mesh allows external clients to connect to a published port on **any node**, even if the container isn't running there.

**How it works:**

- `docker service create -p 80:80` opens port 80 on all nodes.

- Docker routes the traffic internally to service replicas.

**Pros:**

- Simplifies ingress.

- No need for external LB.

**Cons:**

- Performance overhead due to routing.

- Can’t fine-tune load balancing.

---

## 3. Host Mode Load Balancing

**Host mode** disables routing mesh. Ports are exposed **only** on nodes where the container is running.

```bash
docker service create \ --name myapp \ --publish mode=host,target=8080,published=8080 \ myapp-image
```

**Pros:**

- Direct access = better performance.

- Can be used with external load balancers.

**Cons:**

- Requires manual load balancing setup.

- Less transparent than VIP.

---

## Common Architecture Patterns

| Scenario                                          | Recommended Mode        | Why                                               |
| ------------------------------------------------- | ----------------------- | ------------------------------------------------- |
| Internal microservices communication              | VIP                     | Simple, automatic DNS + load balancing            |
| External access to a service from outside Swarm   | Routing Mesh            | No need for external LB, handled by Docker        |
| High-performance, edge-level HTTP routing         | Host Mode + External LB | Better control, avoids internal routing           |
| Integrating with cloud-native LBs (e.g., AWS ALB) | Host Mode               | IP-based targets require container-local exposure |

---

## Changing Load Balancing Algorithm

Docker Swarm does **not allow** configuring the IPVS algorithm (always round-robin). To use `leastconn`, `source`, or sticky sessions, use **host mode** and an **external LB** like:

- HAProxy

- Nginx

- Envoy

- Traefik

---

## Recommendation

If you need anything beyond **simple round-robin**:

| Use Case                        | Recommendation                                              |
| ------------------------------- | ----------------------------------------------------------- |
| Web traffic                     | Use **Traefik** or **Nginx** as reverse proxy/load balancer |
| L4 balancing (TCP/UDP)          | Use **HAProxy**, **Envoy**, or **MetalLB**                  |
| Need sticky sessions            | External LB with cookie/header affinity                     |
| Want fine-grained routing logic | Use **Traefik + labels**, or **Kubernetes Ingress/Service** |

---

# HAProxy + Docker Swarm – Minimal Example

## Folder Structure

```bash
haproxy-swarm-demo/
├── docker-compose.yml
├── haproxy/
│   └── haproxy.cfg
```

---

## haproxy/haproxy.cfg

```bash
global
    daemon
    maxconn 256

defaults
    mode http
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend http-in
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin  # Change to leastconn if needed
    server web1 web1:8080 check
    server web2 web2:8080 check
    server web3 web3:8080 check
```

---

## docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    image: nginx
    deploy:
      replicas: 3
      placement:
        constraints: [node.role == worker]
    networks:
      - webnet
    ports:
      - target: 80
        published: 8080
        protocol: tcp
        mode: host

  haproxy:
    image: haproxy:alpine
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    networks:
      - webnet
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  webnet:
```

---

## Deploy Steps

```bash
docker swarm init docker stack deploy -c docker-compose.yml haproxy-demo
```

Access the service at: `http://<manager-node-ip>:80`

---

## Customize Load Balancing Algorithm

Change in `haproxy.cfg`:

```bash
balance leastconn # Or 'source' for sticky sessions
```



---

## Validation

```bash
curl http://<manager-node-ip> docker service logs haproxy-demo_haproxy
```
