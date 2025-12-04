# Docker DNS Complete Guide

## Table of Contents
1. [DNS Basics: How It Works](#dns-basics)
2. [Single Network DNS](#single-network-dns)
3. [Cross-Network Communication](#cross-network-communication)
4. [Docker Compose DNS](#docker-compose-dns)
5. [Troubleshooting DNS](#troubleshooting-dns)

---

## 1. DNS Basics: How It Works

### The Foundation: resolv.conf and DNS Servers

Every Linux container has a `/etc/resolv.conf` file that tells it where to send DNS queries:

```bash
docker network create my-net
docker run -d --name web --network my-net nginx

# Check DNS configuration
docker exec web cat /etc/resolv.conf
# Output:
# nameserver 127.0.0.11
# options ndots:0
```

**What is 127.0.0.11?**
- A special "magic" IP address
- NOT a real IP inside the container
- Docker intercepts queries to this IP
- Redirects them to the centralized DNS server in Docker daemon

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    HOST MACHINE                          │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         Docker Daemon (dockerd)                │    │
│  │                                                 │    │
│  │  ┌──────────────────────────────────────────┐ │    │
│  │  │  DNS Server for "my-net"                 │ │    │
│  │  │  Listens for queries to 127.0.0.11:53    │ │    │
│  │  │                                           │ │    │
│  │  │  In-Memory Database:                     │ │    │
│  │  │  ┌────────────────────────────┐          │ │    │
│  │  │  │ web  → 172.18.0.2          │          │ │    │
│  │  │  │ api  → 172.18.0.3          │          │ │    │
│  │  │  │ db   → 172.18.0.4          │          │ │    │
│  │  │  └────────────────────────────┘          │ │    │
│  │  └──────────────────┬───────────────────────┘ │    │
│  └───────────────────────┼─────────────────────────┘    │
└─────────────────────────┼──────────────────────────────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
       ┌────▼────┐   ┌────▼────┐   ┌───▼────┐
       │Container│   │Container│   │Container│
       │  web    │   │  api    │   │   db    │
       │         │   │         │   │         │
       │resolv.  │   │resolv.  │   │resolv.  │
       │conf:    │   │conf:    │   │conf:    │
       │127.0.0.11│  │127.0.0.11│  │127.0.0.11│
       └─────────┘   └─────────┘   └─────────┘
```

---

## 2. Single Network DNS

### How DNS Works in One Network

**Scenario**: Three containers in the same network

```bash
# Create network and containers
docker network create shop-net
docker run -d --name frontend --network shop-net nginx
docker run -d --name backend --network shop-net node
docker run -d --name database --network shop-net postgres
```

**What Happens Internally:**

```
Docker Daemon creates ONE DNS server for "shop-net"

DNS Database:
┌──────────────────────────┐
│ frontend  → 172.19.0.2   │
│ backend   → 172.19.0.3   │
│ database  → 172.19.0.4   │
└──────────────────────────┘
```

### DNS Resolution Flow

**Example**: frontend wants to call backend

```
Step 1: Application Code
┌─────────────────────────────────┐
│ Container: frontend             │
│                                 │
│ fetch('http://backend:3000')    │
└────────────┬────────────────────┘
             │
Step 2: Check resolv.conf
             │
             ▼
┌─────────────────────────────────┐
│ Read /etc/resolv.conf           │
│ Found: nameserver 127.0.0.11    │
└────────────┬────────────────────┘
             │
Step 3: Send DNS Query
             │
             ▼
   Query: "What is IP of 'backend'?"
   To: 127.0.0.11:53
             │
             ▼
┌─────────────────────────────────────────┐
│ Docker Daemon intercepts query          │
│ Looks up in shop-net DNS database       │
│ Found: backend → 172.19.0.3             │
└────────────┬────────────────────────────┘
             │
Step 4: Return DNS Response
             │
             ▼
┌─────────────────────────────────┐
│ Container: frontend             │
│ Received: backend = 172.19.0.3  │
│ Now connects to 172.19.0.3:3000 │
└─────────────────────────────────┘
```

### Testing DNS Resolution

```bash
# Test from frontend
docker exec frontend nslookup backend
# Server:    127.0.0.11
# Address:   127.0.0.11:53
# 
# Name:      backend
# Address:   172.19.0.3

# Test reverse (backend to frontend)
docker exec backend nslookup frontend
# Name:      frontend
# Address:   172.19.0.2

# Both directions work! ✓
```

### What's NOT in /etc/hosts

```bash
# Check /etc/hosts in frontend
docker exec frontend cat /etc/hosts
# Output:
# 127.0.0.1       localhost
# 172.19.0.2      a3b5c7 frontend  ← Only itself!
# 
# No "backend" or "database" here!

# They're resolved via DNS, not /etc/hosts
```

---

## 3. Cross-Network Communication

### The Problem: Isolated Networks

By default, containers in different networks **cannot** communicate:

```bash
# Create two separate networks
docker network create frontend-net
docker network create backend-net

# Put containers in different networks
docker run -d --name web --network frontend-net nginx
docker run -d --name api --network backend-net node
```

**Network Isolation:**

```
┌────────────────────────────────────────────────────────┐
│ Docker Daemon                                           │
│                                                         │
│  ┌─────────────────────┐    ┌─────────────────────┐   │
│  │ DNS for             │    │ DNS for             │   │
│  │ "frontend-net"      │    │ "backend-net"       │   │
│  │                     │    │                     │   │
│  │ Database:           │    │ Database:           │   │
│  │ web → 172.20.0.2    │    │ api → 172.21.0.2    │   │
│  └──────────┬──────────┘    └──────────┬──────────┘   │
└─────────────┼──────────────────────────┼──────────────┘
              │                          │
       ┌──────▼──────┐            ┌──────▼──────┐
       │ Container   │            │ Container   │
       │    web      │   ✗✗✗✗    │    api      │
       │ 172.20.0.2  │ ISOLATED  │ 172.21.0.2  │
       └─────────────┘            └─────────────┘
```

**Test the isolation:**

```bash
# Try to ping from web to api
docker exec web ping api
# Error: ping: api: Name or service not known

# Try by IP
docker exec web ping 172.21.0.2
# Error: Network is unreachable (routing prevented)
```

### Solution: Connect Container to Multiple Networks

To enable communication, connect one container to BOTH networks:

```bash
# web is already in frontend-net
# Now also connect it to backend-net
docker network connect backend-net web
```

**What Changed?**

```
Before:
Container "web" has 1 network interface:
- eth0: 172.20.0.2 (frontend-net)

After docker network connect:
Container "web" has 2 network interfaces:
- eth0: 172.20.0.2 (frontend-net)
- eth1: 172.21.0.3 (backend-net)  ← New interface!
```

**Verify:**

```bash
# Check web's interfaces
docker exec web ip addr show
# Output:
# 1: lo: <LOOPBACK,UP>
# 2: eth0: <BROADCAST,UP>
#    inet 172.20.0.2/16  ← frontend-net
# 3: eth1: <BROADCAST,UP>
#    inet 172.21.0.3/16  ← backend-net (NEW!)
```

### DNS Changes When Connected to Multiple Networks

**Critical Understanding**: Web's `/etc/resolv.conf` now has MULTIPLE nameservers!

```bash
# Check resolv.conf after connecting to both networks
docker exec web cat /etc/resolv.conf
# Output:
# nameserver 127.0.0.11  ← DNS for frontend-net
# nameserver 127.0.0.11  ← DNS for backend-net
# 
# Wait... same IP? Yes, but Docker routes to BOTH DNS servers!
```

**How Docker Handles Multiple DNS Servers:**

```
┌─────────────────────────────────────────────────────────┐
│ Container "web" sends DNS query for "api"               │
│ Query goes to: 127.0.0.11                               │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ Docker Daemon receives query                            │
│ Checks: Which networks is "web" connected to?           │
│ Answer: frontend-net AND backend-net                    │
│                                                          │
│ Searches BOTH DNS databases:                            │
│                                                          │
│ 1. Check frontend-net DNS:                              │
│    web → 172.20.0.2  (no "api" found)                   │
│                                                          │
│ 2. Check backend-net DNS:                               │
│    api → 172.21.0.2  ✓ FOUND!                           │
│                                                          │
│ Returns: api = 172.21.0.2                               │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ Container "web" receives response                       │
│ Connects to 172.21.0.2 via eth1 interface               │
└─────────────────────────────────────────────────────────┘
```

### Complete Example: Cross-Network Communication

```bash
# Setup
docker network create frontend-net
docker network create backend-net
docker network create database-net

# Create containers
docker run -d --name web --network frontend-net nginx
docker run -d --name api --network backend-net node
docker run -d --name db --network database-net postgres

# Initial state: All isolated
docker exec web ping api    # FAILS ✗
docker exec api ping db      # FAILS ✗

# Connect web to backend-net
docker network connect backend-net web

# Now web can reach api!
docker exec web ping api     # SUCCESS ✓
docker exec web nslookup api
# Name: api
# Address: 172.21.0.2

# But api still cannot reach db (not connected to database-net)
docker exec api ping db      # FAILS ✗

# Connect api to database-net
docker network connect database-net api

# Now api can reach db!
docker exec api ping db      # SUCCESS ✓
docker exec api nslookup db
# Name: db
# Address: 172.22.0.2

# But web CANNOT reach db (web not in database-net)
docker exec web ping db      # FAILS ✗
```

**Network Topology After Connections:**

```
┌──────────┐
│   web    │───────────────┐
│172.20.0.2│               │
│172.21.0.3│◄──┐           │ frontend-net
└──────────┘   │           │
               │           │
               │           │
┌──────────┐   │ backend-net
│   api    │◄──┤           
│172.21.0.2│   │           
│172.22.0.3│◄──┼───────────┐
└──────────┘   │           │ database-net
               │           │
┌──────────┐   │           │
│    db    │◄──┘           
│172.22.0.2│               
└──────────┘               

web can resolve:  api ✓   db ✗
api can resolve:  web ✗   db ✓
db  can resolve:  web ✗   api ✗
```

### Which DNS Databases Changed?

```bash
# Before connecting web to backend-net
docker network inspect backend-net --format='{{json .Containers}}' | jq
{
  "api-container-id": {
    "Name": "api",
    "IPv4Address": "172.21.0.2/16"
  }
}

# After docker network connect backend-net web
docker network inspect backend-net --format='{{json .Containers}}' | jq
{
  "api-container-id": {
    "Name": "api",
    "IPv4Address": "172.21.0.2/16"
  },
  "web-container-id": {
    "Name": "web",
    "IPv4Address": "172.21.0.3/16"  ← NEW ENTRY!
  }
}

# backend-net DNS database now knows about "web"!
# Now api can also resolve "web"
docker exec api nslookup web
# Name: web
# Address: 172.21.0.3  ← Works!
```

**Summary of DNS Changes:**
1. ✅ **backend-net DNS** adds entry: `web → 172.21.0.3`
2. ✅ **web container** gets new interface: `eth1` with IP `172.21.0.3`
3. ✅ **web's DNS queries** now search both frontend-net AND backend-net databases
4. ❌ **frontend-net DNS** unchanged (still only knows about web)

---

## 4. Docker Compose DNS

### Automatic Network Creation

Docker Compose automatically creates ONE network for all services:

```yaml
# docker-compose.yml
version: '3'
services:
  frontend:
    image: nginx
  backend:
    image: node:alpine
  database:
    image: postgres
```

```bash
docker-compose up -d

# Compose creates: projectname_default network
docker network ls
# NETWORK ID     NAME                  DRIVER
# abc123         myapp_default         bridge

# All services in ONE network, ONE DNS database
docker network inspect myapp_default --format='{{json .Containers}}' | jq
{
  "frontend-id": {
    "Name": "myapp_frontend_1",
    "IPv4Address": "172.23.0.2/16"
  },
  "backend-id": {
    "Name": "myapp_backend_1",
    "IPv4Address": "172.23.0.3/16"
  },
  "database-id": {
    "Name": "myapp_database_1",
    "IPv4Address": "172.23.0.4/16"
  }
}

# DNS Database for myapp_default:
# frontend → 172.23.0.2
# backend  → 172.23.0.3
# database → 172.23.0.4
```

### Service Name Resolution

```bash
# All services can reach each other by service name
docker-compose exec frontend ping backend    # Works!
docker-compose exec backend curl http://database:5432  # Works!

# Check DNS from frontend
docker-compose exec frontend nslookup backend
# Server:    127.0.0.11
# Name:      backend
# Address:   172.23.0.3
```

### Multiple Compose Projects = Isolated DNS

```bash
# Project 1: app1/docker-compose.yml
version: '3'
services:
  api:
    image: nginx

# Project 2: app2/docker-compose.yml
version: '3'
services:
  api:
    image: nginx

# Start both
cd app1 && docker-compose up -d
cd app2 && docker-compose up -d

# Two SEPARATE networks with SEPARATE DNS
docker network ls
# app1_default  ← DNS knows: api → 172.24.0.2
# app2_default  ← DNS knows: api → 172.25.0.2

# They don't conflict! Isolated.
```

### Custom Networks in Compose

You can create multiple networks in one Compose file:

```yaml
version: '3'
services:
  web:
    image: nginx
    networks:
      - frontend
  
  api:
    image: node
    networks:
      - frontend  # Connected to both!
      - backend
  
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
  backend:
```

**DNS Architecture:**

```
┌─────────────────────────────────────────────────────┐
│ Docker Daemon                                        │
│                                                      │
│  ┌───────────────────┐    ┌───────────────────┐    │
│  │ DNS for           │    │ DNS for           │    │
│  │ "frontend" net    │    │ "backend" net     │    │
│  │                   │    │                   │    │
│  │ web → 172.26.0.2  │    │ api → 172.27.0.2  │    │
│  │ api → 172.26.0.3  │    │ db  → 172.27.0.3  │    │
│  └─────────┬─────────┘    └─────────┬─────────┘    │
└────────────┼──────────────────────────┼─────────────┘
             │                          │
    ┌────────▼────────┐        ┌────────▼────────┐
    │  web            │        │  db             │
    │  (frontend)     │        │  (backend)      │
    └─────────────────┘        └─────────────────┘
                │                      │
                └──────┬───────────────┘
                       │
                ┌──────▼──────┐
                │  api        │
                │  (both nets)│
                └─────────────┘

web can resolve:  api ✓   db ✗
api can resolve:  web ✓   db ✓
db  can resolve:  web ✗   api ✓
```

```bash
# Start compose
docker-compose up -d

# Verify DNS
docker-compose exec web nslookup api    # Works!
docker-compose exec web nslookup db     # FAILS (different network)
docker-compose exec api nslookup db     # Works!
docker-compose exec api nslookup web    # Works!
```

---

## 5. Troubleshooting DNS

### Common DNS Commands

```bash
# Check resolv.conf
docker exec <container> cat /etc/resolv.conf

# Test DNS resolution
docker exec <container> nslookup <service-name>
docker exec <container> dig <service-name> +short
docker exec <container> getent hosts <service-name>

# Check if DNS server is responding
docker exec <container> nslookup 127.0.0.11

# See all network interfaces
docker exec <container> ip addr show

# Check which networks container is in
docker inspect <container> --format='{{json .NetworkSettings.Networks}}' | jq

# See all containers in a network
docker network inspect <network-name>

# See DNS database for a network
docker network inspect <network-name> --format='{{json .Containers}}' | jq
```

### DNS Troubleshooting Checklist

**Problem**: Container cannot resolve service name

```bash
# 1. Verify both containers are in the same network
docker inspect web --format='{{json .NetworkSettings.Networks}}' | jq
docker inspect api --format='{{json .NetworkSettings.Networks}}' | jq

# 2. Check if using default bridge (no DNS!)
docker inspect web --format='{{.HostConfig.NetworkMode}}'
# If output is "default" or "bridge" → Problem! Use custom network.

# 3. Verify DNS server is set correctly
docker exec web cat /etc/resolv.conf
# Should see: nameserver 127.0.0.11

# 4. Test if DNS server responds
docker exec web nslookup 127.0.0.11
# Should work

# 5. Check if target container exists in network
docker network inspect <network-name> | grep -A5 "api"

# 6. Try resolution
docker exec web nslookup api

# 7. If still fails, check container logs
docker logs web
docker logs api
```

### Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| `Name or service not known` | Containers in different networks | Use `docker network connect` |
| `server can't find <name>` | Using default bridge | Create custom network |
| No DNS (external sites don't work) | DNS misconfigured | Check `resolv.conf` |
| Works by IP, not name | Wrong network type | Use custom bridge network |
| Intermittent resolution failures | Container restarted with new IP | DNS should auto-update, check Docker version |

---

## Key Takeaways

✅ **One DNS server per network** - Centralized in Docker daemon, not in containers  
✅ **127.0.0.11 is magic** - Docker intercepts queries to this special IP  
✅ **DNS database in memory** - Stored in Docker daemon, not in /etc/hosts  
✅ **Cross-network needs connection** - Use `docker network connect`  
✅ **Connected to multiple networks** - Container queries ALL network DNS databases  
✅ **DNS auto-updates** - When containers join/leave, database updates immediately  
✅ **Compose creates one network** - All services share DNS automatically  
✅ **Default bridge has NO DNS** - Always use custom networks in production  

**Pro Tip**: Think of Docker networks like WiFi networks. Each has its own "router" (DNS server). To talk across networks, you need to connect to both, just like connecting to multiple WiFi networks!
