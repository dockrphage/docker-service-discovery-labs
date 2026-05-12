Below is a **progressive, hands-on lab** designed to teach Docker DNS and service discovery from a DevOps perspective.  
Each stage builds on the previous one, ending with a production-like multi-service scenario.

---

## Lab Overview

**Goal**  
Understand how containers discover each other using:  
- Default bridge network (legacy, no DNS)  
- User-defined bridge network (embedded DNS)  
- Docker Compose service naming  
- Scale & load balancing with DNS round-robin

**Tools**  
Docker Engine, Docker Compose, `dig`, `curl`, `busybox`, `nginx`, `redis`

**Duration** ~90 minutes

---

## Step 1: Baseline – No DNS on Default Bridge

**Objective**  
Show that containers on the default `bridge` network cannot reach each other by name.

```bash
# Run two containers on default bridge
docker run -d --name c1 --rm alpine sleep 3600
docker run -d --name c2 --rm alpine sleep 3600

# Try to ping by name from c2
docker exec c2 ping -c 2 c1
# Expected: ping: bad address 'c1'
```

**Key learning**  
Default bridge → no automatic DNS resolution. Must use `--link` (deprecated) or IP addresses.

```bash
# Clean up
docker stop c1 c2
```

---

## Step 2: User-Defined Bridge – Automatic DNS

**Objective**  
Enable DNS using a custom bridge network.

```bash
# Create network
docker network create mynet

# Run containers on mynet
docker run -d --name app1 --network mynet --rm alpine sleep 3600
docker run -d --name app2 --network mynet --rm alpine sleep 3600

# DNS resolution works
docker exec app2 ping -c 2 app1
# Expected: 64 bytes from ...
```

**Bonus**  
Try resolving by container ID prefix, service name, and even IP reverse lookup.

```bash
docker exec app2 nslookup app1
docker exec app2 cat /etc/resolv.conf   # points to 127.0.0.11 (Docker DNS)
```

---

## Step 3: Multiple Containers with Same Alias

**Objective**  
Learn that Docker DNS returns **all** IPs of containers with the same name (though names must be unique).  
For multiple replicas, we use **service discovery via network aliases** (or later, Compose).

```bash
docker network create lbnet

docker run -d --name web1 --network lbnet --network-alias web --rm nginx
docker run -d --name web2 --network lbnet --network-alias web --rm nginx

# Query DNS alias "web" from a client
docker run --rm --network lbnet appropriate/curl dig +short web
# Expected: two IP addresses (round‑robin order)
```

**Observation**  
Docker does **not** load balance – DNS returns both IPs. The client picks one (typically first).

---

## Step 4: Docker Compose – Service Names as DNS

**Objective**  
Use Compose where service names become resolvable across containers.

`docker-compose.yml` (v3):
```yaml
version: '3'
services:
  redis:
    image: redis:alpine
  app:
    image: alpine
    command: sh -c "apk add --no-cache bind-tools && sleep 3600"
    depends_on:
      - redis
```

```bash
docker-compose up -d
docker-compose exec app nslookup redis
# Success – "redis" resolves to container IP
```

**Key learning**  
Compose creates a default network. Service name = DNS name.

---

## Step 5: DNS Across Multiple Compose Projects

**Objective**  
Show service discovery between different Compose apps using external networks.

**Project A** (`backend/`):
```yaml
version: '3'
services:
  api:
    image: nginx
    networks:
      - shared
networks:
  shared:
    external: true
```

**Project B** (`frontend/`):
```yaml
version: '3'
services:
  web:
    image: alpine
    command: sleep 3600
    networks:
      - shared
networks:
  shared:
    external: true
```

```bash
docker network create shared
cd backend && docker-compose up -d
cd ../frontend && docker-compose up -d
docker-compose exec web ping api   # works
```

---

## Step 6: Scale & DNS Round‑Robin

**Objective**  
See how Docker DNS returns multiple IPs for the same service name when scaled.

```yaml
version: '3'
services:
  whoami:
    image: containous/whoami
    deploy:
      replicas: 3   # Swarm mode
  client:
    image: alpine
    command: sh -c "apk add --no-cache bind-tools curl && sleep 3600"
```

*Note: Scaling replicas works fully in Swarm. For plain Compose, use `docker-compose up --scale whoami=3` (v2.4+).*

```bash
docker-compose up --scale whoami=3 -d
docker-compose exec client dig +short whoami
# Returns 3 IPs, order shuffled on each DNS query
```

**Test load distribution**  
Send multiple requests – observe requests hit different IPs.

```bash
for i in {1..6}; do docker-compose exec client curl -s whoami | grep "Hostname"; done
```

---

## Step 7: Custom DNS Entry & `extra_hosts`

**Objective**  
Sometimes you need to override DNS (e.g., point `db.local` to a specific container).

```yaml
services:
  app:
    image: alpine
    command: sleep 3600
    extra_hosts:
      - "db.local:192.168.1.100"
      - "host.docker.internal:host-gateway"
```

Test:
```bash
docker-compose exec app ping db.local
```

---

## Step 8: Debugging DNS Issues (DevOps skill)

Check:
1. `/etc/resolv.conf` – should show `nameserver 127.0.0.11`
2. `nslookup` / `dig` queries
3. Network isolation – containers must share at least one network
4. Docker daemon logs (`journalctl -u docker | grep DNS`)

```bash
# Inspect a container's DNS config
docker inspect <container> | jq '.[].NetworkSettings.DNS'

# Force custom DNS in daemon.json
{
  "dns": ["8.8.8.8", "1.1.1.1"]
}
```

---

## Step 9: Final Capstone – Multi‑Service App

**Scenario**  
- `frontend` (nginx) talks to `backend` (nodejs) via DNS name `backend`.
- `backend` connects to `redis` via DNS name `redis`.
- Both backend & redis scaled to 2 replicas.

**docker-compose.yml**:
```yaml
version: '3.8'
services:
  frontend:
    image: nginx:alpine
    depends_on:
      - backend
    environment:
      - BACKEND_URL=http://backend:3000

  backend:
    image: your-node-app
    deploy:
      replicas: 2
    depends_on:
      - redis

  redis:
    image: redis:alpine
    deploy:
      replicas: 2
```

**Validation**  
```bash
# From frontend, resolve backend
docker-compose exec frontend nslookup backend

# Simulate redis failover – one redis container stops, DNS updates TTL (default 60s)
```

---

## Summary Table – What You Learned

| Step | Concept | DNS behavior |
|------|---------|---------------|
| 1 | Default bridge | No name resolution |
| 2 | Custom bridge | Automatic DNS via embedded server |
| 3 | Network aliases | Multiple IPs per alias |
| 4 | Compose | Service names = DNS names |
| 5 | Cross‑project | Need shared external network |
| 6 | Scaling | DNS round‑robin (client side) |
| 7 | Overrides | `extra_hosts` bypasses DNS |
| 8 | Debugging | Check resolv.conf & nslookup |

---

## Next Steps for Production

- Integrate with **Consul** or **etcd** for advanced service mesh.
- Use **Traefik** or **nginx‑upstream** for server‑side load balancing.
- Explore **Docker Swarm’s VIP** (Virtual IP) vs DNS RR.
