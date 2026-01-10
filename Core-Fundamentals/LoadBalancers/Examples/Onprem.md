# On-Premises Load Balancer Examples

---

## Hardware Load Balancers

### 1. F5 BIG-IP

* Enterprise-grade hardware appliance
* **Layer:** L4-L7 (full stack)
* **Algorithms:** All algorithms mentioned above

**Features:**

* Advanced SSL/TLS offloading with hardware acceleration
* Full proxy architecture (L7)
* iRules (Tcl scripting for custom logic)
* Application firewall (WAF)
* DDoS protection

**Use Case:** Large enterprises, financial institutions, critical infrastructure
**Capacity:** Up to 1.6 Tbps throughput, millions of concurrent connections
**Cost:** $20K-$200K+ per appliance

**Configuration Example:**

```tcl
# iRule for custom routing
when HTTP_REQUEST {
    if { [HTTP::uri] starts_with "/api" } {
        pool api_server_pool
    } elseif { [HTTP::uri] starts_with "/images" } {
        pool cdn_pool
    } else {
        pool default_web_pool
    }
}
```

---

### 2. Citrix ADC (NetScaler)

* Another enterprise hardware solution
* **Layer:** L4-L7
* **Algorithms:** All standard algorithms plus proprietary ones
* **Specialty:** Application delivery optimization, WAN optimization
* **Use Case:** Enterprises with Citrix Virtual Apps deployments

---

### 3. A10 Networks Thunder

* High-performance hardware load balancer
* **Layer:** L4-L7
* **Specialty:** DDoS protection, carrier-grade NAT
* **Use Case:** Telecom providers, large enterprises

---

## Software Load Balancers

### 1. HAProxy (High Availability Proxy)

* Open-source, industry standard
* **Layer:** L4 (TCP mode) and L7 (HTTP mode)
* **Algorithms:** Round robin, weighted round robin, least connections, source IP hash, URI hash, consistent hashing
* **Deployment:** Linux servers, VMs, containers

**Configuration Example:**

```haproxy
global
    maxconn 50000
    log 127.0.0.1 local0

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend web_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/mycert.pem
    
    # L7 content-based routing
    acl is_api path_beg /api
    acl is_images path_beg /images
    
    use_backend api_servers if is_api
    use_backend image_servers if is_images
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /health
    server web1 10.0.1.10:8080 check
    server web2 10.0.1.11:8080 check
    server web3 10.0.1.12:8080 check

backend api_servers
    balance leastconn
    option httpchk GET /api/health
    server api1 10.0.2.10:8081 check
    server api2 10.0.2.11:8081 check

backend image_servers
    balance uri
    hash-type consistent
    server img1 10.0.3.10:8082 check
    server img2 10.0.3.11:8082 check
```

**When to Use:**

* Budget-conscious deployments
* High-performance requirements (can handle 40Gbps+ with proper hardware)
* Need full control and customization
* Kubernetes ingress controllers

---

### 2. NGINX (Plus and Open Source)

* NGINX Open Source: Basic L7 load balancing
* NGINX Plus: Commercial version with advanced features
* **Layer:** Primarily L7 (HTTP/HTTPS), L4 via stream module
* **Algorithms:** Round robin, least connections, IP hash, generic hash, least time (Plus), random with two choices

**Configuration Example:**

```nginx
upstream backend_servers {
    least_conn;  # Algorithm
    
    server 10.0.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 weight=2;
    server 10.0.1.12:8080 weight=1 backup;  # Backup server
    
    # Session persistence
    sticky cookie srv_id expires=1h domain=.example.com path=/;
}

server {
    listen 80;
    listen 443 ssl http2;
    
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    # L7 routing
    location /api/ {
        proxy_pass http://api_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    location /images/ {
        proxy_pass http://image_servers;
        proxy_cache my_cache;
        proxy_cache_valid 200 1h;
    }
    
    location / {
        proxy_pass http://backend_servers;
        health_check interval=5s fails=3 passes=2;
    }
}
```

**When to Use:**

* Web application load balancing
* Need reverse proxy + load balancing + caching
* Kubernetes ingress
* Microservices architectures

---

### 3. Linux Virtual Server (LVS/IPVS)

* Kernel-level L4 load balancer built into Linux
* **Layer:** L4 only
* **Algorithms:** Round robin, weighted round robin, least connections, weighted least connections, source IP hash, destination IP hash
* **Performance:** Extremely high (kernel-level, minimal overhead)

**Modes:**

* NAT: Load balancer changes packet IP
* Direct Routing (DR): Servers respond directly to clients
* IP Tunneling: Encapsulates packets for routing

**When to Use:**

* Maximum L4 performance
* Simple load distribution sufficient
* Cost-sensitive (free)

---

### 4. Traefik

* Modern, cloud-native load balancer
* **Layer:** L7
* **Specialty:** Automatic service discovery, dynamic configuration
* **Integration:** Kubernetes, Docker, Consul, Etcd

**Configuration Example (Docker):**

```yaml
version: '3'

services:
  traefik:
    image: traefik:v2.9
    command:
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  webapp:
    image: myapp:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.webapp.rule=Host(`example.com`)"
      - "traefik.http.services.webapp.loadbalancer.server.port=8080"
    deploy:
      replicas: 3  # Traefik automatically load balances across replicas
```

**When to Use:**

* Container-based deployments
* Microservices with dynamic scaling
* Need automatic configuration

