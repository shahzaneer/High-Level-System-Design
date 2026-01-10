# GCP Load Balancer Examples

---

## 1. Global HTTP(S) Load Balancer

* **Layer:** L7 (HTTP/HTTPS)
* **Algorithms:**

  * Round robin (default)
  * Rate-based (sends traffic based on requests per second capacity)
  * Utilization-based (based on target CPU utilization)

**Key Features:**

* Truly global: Anycast IP, traffic enters nearest Google PoP
* Global load balancing: Distributes traffic across regions automatically
* Intelligent routing: Sends users to nearest healthy backend
* URL maps: Path-based and host-based routing
* Cloud CDN integration: Native caching
* Cloud Armor integration: DDoS and WAF protection
* SSL/TLS: Managed certificates, custom certificates, SSL policies
* Backend services: Can mix instance groups, NEGs (Network Endpoint Groups), Cloud Storage buckets, Cloud Run services

**Configuration Example (gcloud):**

```bash
# Create instance groups in multiple regions
gcloud compute instance-groups managed create web-us \
    --region us-central1 \
    --template web-template \
    --size 3

gcloud compute instance-groups managed create web-eu \
    --region europe-west1 \
    --template web-template \
    --size 3

# Create health check
gcloud compute health-checks create http web-health-check \
    --port 80 \
    --request-path /health

# Create backend service
gcloud compute backend-services create web-backend \
    --protocol HTTP \
    --health-checks web-health-check \
    --global

# Add backends (instance groups)
gcloud compute backend-services add-backend web-backend \
    --instance-group web-us \
    --instance-group-region us-central1 \
    --balancing-mode UTILIZATION \
    --max-utilization 0.8 \
    --global

gcloud compute backend-services add-backend web-backend \
    --instance-group web-eu \
    --instance-group-region europe-west1 \
    --balancing-mode UTILIZATION \
    --max-utilization 0.8 \
    --global

# Create URL map (routing rules)
gcloud compute url-maps create web-map \
    --default-service web-backend

# Add path-based routing
gcloud compute url-maps add-path-matcher web-map \
    --path-matcher-name api-matcher \
    --default-service web-backend \
    --path-rules "/api/*=api-backend,/images/*=image-backend"

# Create target HTTP(S) proxy
gcloud compute target-https-proxies create web-proxy \
    --url-map web-map \
    --ssl-certificates my-cert

# Create global forwarding rule
gcloud compute forwarding-rules create web-lb \
    --global \
    --target-https-proxy web-proxy \
    --ports 443
```

**Unique Feature:** Cross-region load balancing

* Users automatically routed to nearest healthy backend. Failover between regions using single global IP.

**When to Use:**

* Global web applications
* Need CDN (Cloud CDN) integration
* Multi-region deployment
* DDoS protection required (Cloud Armor)
* Serverless backends (Cloud Run, Cloud Functions)

**Real-World Scenario:**

* Global SaaS application with backends in `us-central1`, `europe-west1`, `asia-southeast1`
* Users routed to nearest region
* Automatic failover if a region goes down
* Cloud CDN caches static assets globally
* Cloud Armor protects against DDoS

---

## 2. Regional External HTTP(S) Load Balancer

* **Layer:** L7
* **Difference from Global:** Operates within a single region, lower latency, supports advanced traffic management (Traffic Director)

**When to Use:**

* Regional applications
* Need advanced traffic management (Traffic Director service mesh)
* Lower cost than global LB

---

## 3. Internal HTTP(S) Load Balancer

* **Layer:** L7
* **What It Is:** Private load balancer, only accessible within VPC. No public IP

**When to Use:**

* Internal microservices
* Multi-tier applications (web tier to app tier)
* Private APIs

**Real-World Scenario:**

* Three-tier application:

  * External HTTPS LB → Web tier (public)
  * Internal HTTPS LB → App tier (private, within VPC)
  * Internal HTTPS LB → Data tier (private, within VPC)

---

## 4. External Network Load Balancer

* **Layer:** L4 (TCP, UDP, SSL)
* **Type:** Regional (not global)
* **Algorithms:**

  * 5-tuple hash (protocol, source IP, source port, dest IP, dest port)
  * Connection tracking

**Key Features:**

* High performance: Millions of queries per second
* Preserve client IP: With direct server return
* SSL proxy: Can terminate SSL while staying L4
* Regional: Single region only

**Configuration Example:**

```bash
# Create instance group
gcloud compute instance-groups managed create game-servers \
    --region us-central1 \
    --template game-template \
    --size 5

# Create health check
gcloud compute health-checks create tcp game-health \
    --port 7777

# Create backend service
gcloud compute backend-services create game-backend \
    --protocol TCP \
    --health-checks game-health \
    --region us-central1

# Add backend
gcloud compute backend-services add-backend game-backend \
    --instance-group game-servers \
    --instance-group-region us-central1 \
    --region us-central1

# Create forwarding rule
gcloud compute forwarding-rules create game-lb \
    --region us-central1 \
    --backend-service game-backend \
    --ports 7777 \
    --protocol TCP
```

**When to Use:**

* Gaming servers (custom protocols)
* Database load balancing
* IoT devices (MQTT)
* Any non-HTTP TCP/UDP traffic

---

## 5. Internal Network Load Balancer

* **Layer:** L4
* **What It Is:** Private L4 load balancer, VPC-internal only

**When to Use:**

* Internal services requiring L4
* Database read replicas
* Internal TCP/UDP services

