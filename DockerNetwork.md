# Docker Networking Guide

## 1. Docker Network Types

### Default Bridge (docker0)
- **What it is**: The default network Docker creates automatically
- **IP Range**: Usually `172.17.0.0/16`
- **Key Limitation**: Containers can only communicate via IP addresses, not names

```bash
# Run containers on default bridge
docker run -d --name app1 nginx
docker run -d --name app2 nginx

# app2 cannot ping "app1" by name - must use IP
docker exec app2 ping 172.17.0.2  # Works
docker exec app2 ping app1         # Fails!
```

**Why no DNS on default bridge?**

Docker's default bridge is a legacy network mode. It was designed before Docker had built-in DNS. Here's what happens:

```bash
# Check DNS resolver in container on default bridge
docker exec app1 cat /etc/resolv.conf
# Output: nameserver 8.8.8.8 (or your host's DNS)

# No Docker internal DNS server!
# Container uses external DNS which knows nothing about "app1" or "app2"
```

On default bridge, Docker:
- ❌ Does NOT run an internal DNS server
- ❌ Does NOT maintain a name-to-IP mapping
- ✅ Only links containers via `/etc/hosts` if you use deprecated `--link` flag

**The Old Way: Using --link (Deprecated)**

Before Docker had DNS, you had to manually link containers:

```bash
# Start database container
docker run -d --name mydb postgres

# Link app to database using --link
docker run -d --name myapp --link mydb:database nginx

# Check /etc/hosts in myapp
docker exec myapp cat /etc/hosts
# Output:
# 127.0.0.1       localhost
# 172.17.0.2      mydb database  ← Docker added this entry!
# 172.17.0.3      abc123 myapp

# Now "mydb" resolves via /etc/hosts
docker exec myapp ping mydb    # Works!
docker exec myapp ping database # Works! (using alias)

# But mydb CANNOT reach myapp (one-way only)
docker exec mydb ping myapp     # Fails!
```

**Problems with --link:**
- ❌ One-way only (myapp → mydb, not reverse)
- ❌ Must specify at container creation
- ❌ Can't link containers after they're running
- ❌ Hardcoded in `/etc/hosts` (not dynamic)
- ❌ If mydb restarts with new IP, myapp's `/etc/hosts` is stale!

**Why custom networks are different:**

```bash
docker network create my-net
docker run -d --name app1 --network my-net nginx

# Check DNS resolver
docker exec app1 cat /etc/resolv.conf
# Output: nameserver 127.0.0.11  ← Docker's embedded DNS!

# Docker runs DNS server at 127.0.0.11 inside each container
# This DNS server knows all container names in the network

# Check /etc/hosts - only has container's own entry
docker exec app1 cat /etc/hosts
# 127.0.0.1       localhost
# 172.18.0.2      abc123 app1  ← Only itself!
# No other containers here!

# Other containers resolved via DNS queries to 127.0.0.11, not /etc/hosts
```

### Custom Bridge Network
- **What it is**: User-defined bridge network
- **Key Benefit**: Built-in DNS - containers can call each other by name
- **Isolation**: Separate from default bridge

```bash
# Create custom network
docker network create my-network

# Run containers on custom network
docker run -d --name web --network my-network nginx
docker run -d --name api --network my-network nginx

# Now name resolution works!
docker exec web ping api           # Works! ✓
docker exec api curl http://web    # Works! ✓
```

**Why service names matter**: In production, IPs change. Using names (`http://api:8080`) keeps your code stable.

### Docker Compose Networks
- **Automatic**: Compose creates a custom bridge network automatically
- **Naming**: Services use their service name from `docker-compose.yml`

```yaml
version: '3'
services:
  web:
    image: nginx
  api:
    image: node:alpine
    command: node server.js

# Docker Compose creates network "myapp_default" automatically
# web can call api at: http://api:3000
# api can call web at: http://web:80
```

```bash
# Containers communicate by service name
docker-compose exec web curl http://api:3000
```

---

## 2. Container-to-Host Communication

### How Containers Reach the Host

Containers can communicate with the host machine through the bridge network. Here's the complete flow:

```bash
# Create a simple web server on host
# Host IP on docker0: 172.17.0.1 (gateway)

# Run container
docker run -d --name test nginx

# From container, reach host
docker exec test ping 172.17.0.1  # Reaches host!

# Or use special DNS name
docker exec test ping host.docker.internal  # Also works (on Mac/Windows)
```

### Complete Traffic Flow: Container → Host

```
┌─────────────────────────────────────────────────────────┐
│ Host Machine                                             │
│                                                          │
│  ┌──────────────┐         ┌─────────────────┐          │
│  │ Application  │         │  docker0 bridge │          │
│  │ listening on │◄────────│   172.17.0.1    │          │
│  │ 172.17.0.1   │         │  (gateway IP)   │          │
│  └──────────────┘         └─────────────────┘          │
│                                   ▲                      │
│                                   │                      │
│                           ┌───────┴──────┐              │
│                           │ vethXXXX     │              │
│                           │ (host side)  │              │
│                           └───────┬──────┘              │
└───────────────────────────────────┼──────────────────────┘
                                    │
                    ════════════════╪════════════════
                    Network Namespace Boundary
                    ════════════════╪════════════════
                                    │
┌───────────────────────────────────┼──────────────────────┐
│ Container Namespace                │                      │
│                           ┌────────▼──────┐              │
│                           │ eth0          │              │
│                           │ (container    │              │
│                           │  side)        │              │
│                           │ 172.17.0.2    │              │
│                           └───────────────┘              │
│                                                          │
│  ┌──────────────┐                                        │
│  │ Application  │───────────────────────────────────────▶│
│  │ makes request│  Step 1: Send to 172.17.0.1           │
│  │ to host      │                                        │
│  └──────────────┘                                        │
└─────────────────────────────────────────────────────────┘
```

**Step-by-Step Flow:**

1. **Container app** sends packet to `172.17.0.1` (docker0 gateway)
2. Packet goes through container's **eth0** interface
3. Crosses **veth pair** (virtual ethernet tunnel)
4. Arrives at **docker0 bridge** on host
5. Bridge forwards to host's network stack
6. **Host application** receives the packet

### How veth Pairs Work

**veth** = Virtual Ethernet - always comes in pairs (like two ends of a cable)

```bash
# See veth pairs
ip link show type veth

# Output example:
# 8: veth7f3a2b1@if7: <BROADCAST,MULTICAST,UP>
#    ↑                  ↑
#    Host side          Container side (if7 = interface 7 in container)

# Inside container
docker exec test ip link show
# 7: eth0@if8: <BROADCAST,MULTICAST,UP>
#    ↑         ↑
#    Container sees "eth0"    Links to host's interface 8
```

**Visual Representation:**

```
Host                        Container
┌─────────┐                ┌─────────┐
│ veth123 │════════════════│  eth0   │
│ (if 8)  │  veth pair     │  (if 7) │
└────┬────┘                └─────────┘
     │
     ├─ Connected to docker0 bridge
```

**How data flows through veth:**

```bash
# Packet enters veth123 on host → immediately appears on eth0 in container
# It's like a tunnel between two network namespaces!

# See this in action
# Terminal 1: Monitor host side
sudo tcpdump -i veth123abc

# Terminal 2: Generate traffic from container
docker exec test ping 8.8.8.8

# You'll see packets on BOTH sides of the veth pair
```

---

## 3. DNS Deep Dive - How Name Resolution Works

---

## 3. DNS Deep Dive - How Name Resolution Works

### Architecture: One Centralized DNS Server Per Network

**Critical Understanding**: Docker runs ONE centralized DNS server per network (not one per container). All containers in that network query the SAME DNS server.

```
┌──────────────────────────────────────────────────────────┐
│ Docker Daemon (dockerd) - runs on HOST                    │
│                                                           │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Centralized DNS Server for "my-network"           │  │
│  │ Location: Host process (not inside containers)    │  │
│  │                                                    │  │
│  │ DNS Database (in memory):                         │  │
│  │  web    → 172.18.0.2                              │  │
│  │  api    → 172.18.0.3                              │  │
│  │  db     → 172.18.0.4                              │  │
│  └─────────────────┬─────────────────────────────────┘  │
│                    │ Listens on: 127.0.0.11:53          │
└────────────────────┼──────────────────────────────────────┘
                     │
         ┌───────────┼───────────┬────────────────┐
         │           │           │                │
    ┌────▼─────┐ ┌──▼──────┐ ┌──▼──────┐   ┌────▼─────┐
    │Container │ │Container│ │Container│   │Container │
    │   web    │ │   api   │ │   db    │   │  cache   │
    │          │ │         │ │         │   │ (added   │
    │resolv.   │ │resolv.  │ │resolv.  │   │  later)  │
    │conf:     │ │conf:    │ │conf:    │   │resolv.   │
    │127.0.0.11│ │127.0.0.11│ │127.0.0.11│  │conf:     │
    └──────────┘ └─────────┘ └─────────┘   │127.0.0.11│
                                            └──────────┘
    All point to SAME centralized DNS server
```

**Key Points:**
- ✅ DNS server runs in Docker daemon (on host), NOT inside containers
- ✅ Each container's `/etc/resolv.conf` points to `127.0.0.11`
- ✅ `127.0.0.11` is a special IP that redirects to the centralized DNS server
- ✅ All containers in the network query the same DNS database

### What Happens When You Add a New Container

```bash
# Create network
docker network create my-network

# Start first container
docker run -d --name web --network my-network nginx

# DNS server now knows:
# web → 172.18.0.2

# Check DNS from web (nobody else exists yet)
docker exec web nslookup api
# Error: server can't find api  ← Expected, api doesn't exist

# Now add second container
docker run -d --name api --network my-network nginx

# DNS server AUTOMATICALLY updates its database:
# web → 172.18.0.2
# api → 172.18.0.3

# NOW web can resolve api immediately!
docker exec web nslookup api
# Server:    127.0.0.11
# Name:      api
# Address:   172.18.0.3  ← Works instantly!

# And api can resolve web too!
docker exec api nslookup web
# Name:      web
# Address:   172.18.0.2  ← Both directions work!
```

**What changed?**
- ❌ NOT the `/etc/resolv.conf` files (they stay the same: `127.0.0.11`)
- ❌ NOT the `/etc/hosts` files (they only contain their own info)
- ✅ ONLY the centralized DNS database in Docker daemon

### Where Are the Mappings Stored?

**Answer: In Docker daemon's memory (not in container files)**

```bash
# The DNS database is NOT stored in files inside containers
# It's stored in Docker daemon's internal state

# You can VIEW the mappings through Docker API:
docker network inspect my-network --format='{{json .Containers}}' | jq

# Output shows the database:
{
  "abc123...": {
    "Name": "web",
    "IPv4Address": "172.18.0.2/16",
    "IPv6Address": ""
  },
  "def456...": {
    "Name": "api",
    "IPv4Address": "172.18.0.3/16",
    "IPv6Address": ""
  }
}

# Docker daemon uses this to answer DNS queries
```

### Complete DNS Resolution Flow

**Scenario**: Container "web" wants to reach "api"

```
┌──────────────────────────────────────────────────────────┐
│ Container "web" (172.18.0.2)                              │
│                                                           │
│  1. Application code: fetch('http://api:3000')           │
│     ↓                                                     │
│  2. OS checks /etc/resolv.conf                           │
│     Found: nameserver 127.0.0.11                         │
│     ↓                                                     │
│  3. Send DNS query: "What is IP of 'api'?"               │
│     Destination: 127.0.0.11:53                           │
└─────────────────────────┬─────────────────────────────────┘
                          │ DNS query packet
                          ↓
┌──────────────────────────────────────────────────────────┐
│ Docker Daemon (Host) - DNS Server for my-network         │
│                                                           │
│  4. Receive query: "What is IP of 'api'?"                │
│     ↓                                                     │
│  5. Lookup in DNS database:                              │
│     Database: {web: 172.18.0.2, api: 172.18.0.3}         │
│     ↓                                                     │
│  6. Found: api → 172.18.0.3                              │
│     ↓                                                     │
│  7. Send DNS response: "api is 172.18.0.3"               │
└─────────────────────────┬─────────────────────────────────┘
                          │ DNS response packet
                          ↓
┌──────────────────────────────────────────────────────────┐
│ Container "web"                                           │
│                                                           │
│  8. Receive DNS response: api = 172.18.0.3               │
│     ↓                                                     │
│  9. Now send HTTP request to 172.18.0.3:3000             │
│     ↓                                                     │
│  10. Connection established!                             │
└──────────────────────────────────────────────────────────┘
```

### Verify DNS is NOT in /etc/hosts

```bash
docker network create demo
docker run -d --name web --network demo nginx
docker run -d --name api --network demo nginx

# Check /etc/hosts in web container
docker exec web cat /etc/hosts
# Output:
# 127.0.0.1       localhost
# 172.18.0.2      abc123 web
# ↑ Only itself! No "api" entry here!

# But web CAN resolve api
docker exec web nslookup api
# Name:      api
# Address:   172.18.0.3  ← Resolved via DNS, not /etc/hosts!

# Proof: Use strace to see DNS query
docker exec web sh -c "apk add strace && strace -e trace=connect ping -c1 api"
# You'll see: connect to 127.0.0.11:53 (DNS query)
```

### Summary: DNS vs /etc/hosts

| Feature | Default Bridge + --link | Custom Network |
|---------|------------------------|----------------|
| **Resolution Method** | `/etc/hosts` file | DNS query to 127.0.0.11 |
| **Storage Location** | Inside each container's `/etc/hosts` | Docker daemon memory |
| **Updates** | Static, set at creation | Dynamic, automatic |
| **Direction** | One-way (--link is directional) | Bidirectional |
| **When IP changes** | Stale entries | Always current |

**Key Takeaway:**
- `/etc/hosts` = Old way, static file inside container
- DNS (127.0.0.11) = Modern way, dynamic centralized database in Docker daemon
- When you add a container, ONLY the centralized DNS database updates
- All containers immediately see the new container (no file changes needed)

### Multiple Networks = Multiple DNS Servers

While each network has ONE centralized DNS server, different networks have SEPARATE DNS servers:

```bash
# Create two networks
docker network create net-a
docker network create net-b

# Run containers
docker run -d --name web-a --network net-a nginx
docker run -d --name web-b --network net-b nginx

# Both containers see DNS at 127.0.0.11, but they're DIFFERENT DNS servers!
docker exec web-a cat /etc/resolv.conf
# nameserver 127.0.0.11  ← net-a's DNS

docker exec web-b cat /etc/resolv.conf
# nameserver 127.0.0.11  ← net-b's DNS (different database!)
```

**Architecture with Multiple Networks:**

```
┌──────────────────────────────────────────────────────────┐
│ Docker Daemon (dockerd)                                   │
│                                                           │
│  ┌─────────────────────┐      ┌─────────────────────┐   │
│  │ DNS for "net-a"     │      │ DNS for "net-b"     │   │
│  │ Database:           │      │ Database:           │   │
│  │ web-a → 172.18.0.2  │      │ web-b → 172.19.0.2  │   │
│  │ api-a → 172.18.0.3  │      │ db-b  → 172.19.0.3  │   │
│  └──────────┬──────────┘      └──────────┬──────────┘   │
│             │                             │               │
└─────────────┼─────────────────────────────┼───────────────┘
              │                             │
      ┌───────▼────────┐            ┌──────▼─────────┐
      │ Container web-a│            │ Container web-b│
      │ resolv.conf:   │            │ resolv.conf:   │
      │ 127.0.0.11     │            │ 127.0.0.11     │
      └────────────────┘            └────────────────┘
      
      web-a can resolve:            web-b can resolve:
      ✅ api-a                       ✅ db-b
      ❌ web-b (different network)   ❌ web-a (different network)
```

### Viewing DNS Records

Docker doesn't use traditional DNS zone files. DNS mappings are stored in Docker's internal database. Here's how to inspect them:

```bash
# Create network with containers
docker network create demo
docker run -d --name web --network demo nginx
docker run -d --name api --network demo nginx
docker run -d --name db --network demo postgres

# Check if names resolve
docker exec web nslookup api
# Output:
# Server:    127.0.0.11
# Address:   127.0.0.11:53
#
# Name:      api
# Address:   172.18.0.3

docker exec web nslookup db
# Name:      db
# Address:   172.18.0.4

# Test with dig (more details)
docker exec web dig api +short
# 172.18.0.3
```

#### Method 2: Inspect Network

```bash
# See all containers and their IPs in a network
docker network inspect demo

# Output shows DNS mappings:
{
  "Containers": {
    "abc123": {
      "Name": "web",
      "IPv4Address": "172.18.0.2/16"
    },
    "def456": {
      "Name": "api",
      "IPv4Address": "172.18.0.3/16"
    },
    "ghi789": {
      "Name": "db",
      "IPv4Address": "172.18.0.4/16"
    }
  }
}

# Docker DNS uses this mapping:
# "api" → 172.18.0.3
# "db" → 172.18.0.4
```

#### Method 3: Check Container's /etc/hosts

```bash
# Note: /etc/hosts only has the container's own entry
docker exec web cat /etc/hosts
# Output:
# 127.0.0.1       localhost
# 172.18.0.2      abc123 web  ← Only itself!

# Other containers resolved via DNS, NOT /etc/hosts
```

### Docker Compose DNS

Each Compose project creates its own network with its own DNS server:

```yaml
# docker-compose.yml
version: '3'
services:
  web:
    image: nginx
  api:
    image: node
  db:
    image: postgres
```

```bash
# Start compose
docker-compose up -d

# Compose creates network: "myproject_default"
docker network ls
# NETWORK ID     NAME                  DRIVER
# abc123         myproject_default     bridge

# Check DNS in compose network
docker-compose exec web nslookup api
# Server:    127.0.0.11
# Name:      api
# Address:   172.19.0.3

docker-compose exec web nslookup db
# Name:      db
# Address:   172.19.0.4

# See all services in compose network
docker network inspect myproject_default
```

**Key Points:**
- ✅ Each compose project = 1 network = 1 DNS server
- ✅ Service names from `docker-compose.yml` become DNS names
- ✅ Multiple compose projects are isolated (separate networks, separate DNS)

```bash
# Two compose projects
# Project 1: app1/docker-compose.yml (service: api)
# Project 2: app2/docker-compose.yml (service: api)

# Both have "api" service, but different networks
docker network ls
# app1_default  ← DNS knows about app1's "api"
# app2_default  ← DNS knows about app2's "api"

# No conflict! Isolated networks, isolated DNS
```

### Testing DNS Resolution Between Networks

```bash
# Create networks
docker network create net-a
docker network create net-b

docker run -d --name web-a --network net-a nginx
docker run -d --name web-b --network net-b nginx

# web-a CANNOT resolve web-b (different network)
docker exec web-a nslookup web-b
# Error: server can't find web-b

# Connect web-a to net-b
docker network connect net-b web-a

# Now web-a can resolve names in BOTH networks!
docker exec web-a nslookup web-b
# Name:      web-b
# Address:   172.19.0.2  ← Success!
```

---

## 4. Service-to-Service Communication Scenarios
```bash
docker network create shop-network

docker run -d --name frontend --network shop-network nginx
docker run -d --name backend --network shop-network node

# frontend → backend
docker exec frontend curl http://backend:3000
```
**How it works**: Docker's internal DNS resolves "backend" to its container IP.

### Scenario 2: Different Networks (Isolated)
```bash
docker network create net-a
docker network create net-b

docker run -d --name service1 --network net-a nginx
docker run -d --name service2 --network net-b nginx

# This FAILS - different networks are isolated
docker exec service1 ping service2  # Cannot reach!
```

### Scenario 3: Connect Container to Multiple Networks
```bash
docker network create frontend-net
docker network create backend-net

docker run -d --name api --network backend-net nginx
docker run -d --name db --network backend-net postgres

docker run -d --name web --network frontend-net nginx

# Connect web to backend-net so it can reach api
docker network connect backend-net web

# Now web can reach both networks
docker exec web curl http://api
```

---

## 5. Network Namespace & Traffic Routing

### What is a Network Namespace?
A **network namespace** is an isolated network stack. Each container gets its own:
- Network interfaces
- IP addresses
- Routing tables
- Firewall rules

```bash
# View namespaces
docker run -d --name test nginx
docker inspect test | grep SandboxKey
# Output: /var/run/docker/netns/abc123

# Enter container's network namespace
sudo nsenter --net=/var/run/docker/netns/abc123 ip addr
```

### Traffic Flow Example

```
Container A (172.18.0.2) → Container B (172.18.0.3)
         ↓
    [veth pair]
         ↓
    [docker0 bridge]
         ↓
    [routing decision]
         ↓
    [veth pair]
         ↓
    Container B receives packet
```

**Step-by-step**:
1. **Container A** sends packet to `172.18.0.3`
2. Packet exits via **veth** (virtual ethernet) interface
3. Arrives at **docker bridge** (like a virtual switch)
4. Bridge checks MAC table, forwards to Container B's veth
5. **Container B** receives packet

### Port Mapping (Host to Container)
```bash
docker run -d -p 8080:80 --name web nginx

# Traffic flow:
# Browser → localhost:8080
#    ↓
# [Host iptables NAT rule]
#    ↓
# Container IP:80 (172.17.0.2:80)
```

Docker creates **iptables** rules:
```bash
sudo iptables -t nat -L -n | grep 8080
# DNAT rule: 0.0.0.0:8080 → 172.17.0.2:80
```

---

## 6. Quick Comparison Table

| Network Type | DNS Resolution | Use Case | Isolation |
|--------------|----------------|----------|-----------|
| Default Bridge | ❌ No | Quick testing | Low |
| Custom Bridge | ✅ Yes | Production apps | Medium |
| Compose Network | ✅ Yes | Multi-container apps | Medium |
| Host Network | N/A | High performance | None |
| None | N/A | Maximum isolation | Complete |

---

## 7. Practical Commands

```bash
# List networks
docker network ls

# Inspect network details
docker network inspect my-network

# See container's network config
docker inspect <container> --format='{{.NetworkSettings.Networks}}'

# Check DNS resolution
docker exec <container> nslookup <service-name>
docker exec <container> dig <service-name> +short

# View DNS server being used
docker exec <container> cat /etc/resolv.conf

# View bridge interfaces on host
ip addr show docker0
ip link show type veth

# Check which veth belongs to which container
docker exec <container> ip link show eth0
# Look for "if<number>" then find matching number on host

# Monitor traffic (requires tcpdump in container)
docker exec <container> tcpdump -i eth0

# Test container-to-host connectivity
docker exec <container> ping 172.17.0.1  # docker0 gateway

# See all containers in a network with IPs
docker network inspect <network-name> --format='{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{println}}{{end}}'
```

---

## 8. Key Takeaways

✅ **Use custom networks** for name-based service discovery  
✅ **Docker Compose** automatically creates custom networks  
✅ **Service names** make your architecture flexible and maintainable  
✅ **Network namespaces** provide isolation between containers  
✅ **veth pairs + bridges** handle internal routing  
✅ **iptables** manages external (host ↔ container) traffic

**Golden Rule**: Never rely on container IPs - always use service names in custom networks!
