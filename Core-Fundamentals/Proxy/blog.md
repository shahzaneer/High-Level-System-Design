# Proxy (Forward & Reverse)

## Introduction
Proxies are fundamental networking intermediaries that sit between clients and servers, enabling security, performance optimization, and architectural flexibility. The concept dates to the early days of networked computing but has evolved into a critical building block for modern distributed systems. Reverse proxies power every major web application—they handle TLS termination, load balancing, caching, rate limiting, and API gateway functionality. Forward proxies control outbound access for enterprises and enable privacy-preserving internet access.

Understanding both proxy types is non-negotiable for solution architects: they determine how traffic flows through your system, where security boundaries exist, and how services communicate with each other and the outside world.

## Definition

A **Proxy** is an intermediary server that sits between clients and destination servers, forwarding requests and responses. Proxies operate at different OSI layers (L4 TCP/UDP or L7 HTTP/HTTPS) and can modify, filter, cache, or route traffic based on configured rules.

- **Forward Proxy**: Sits between internal clients and the external internet. Clients initiate connections through the proxy to reach external destinations. The proxy hides the client's identity from the destination server.
- **Reverse Proxy**: Sits between external clients and internal servers. Clients connect to the proxy thinking it's the actual server. The proxy routes requests to backend servers and returns responses. The proxy hides server topology from clients.

## Concept Explanation

### Forward Proxy (Outbound)

```
[Internal Clients] ──→ [Forward Proxy] ──→ [Internet / External Servers]
```

The forward proxy acts as an outbound gateway. Key functions:

**Client Anonymity**: External servers see the proxy's IP, not the client's. Critical for privacy, web scraping, and accessing geo-restricted content.

**Access Control**: Organizations enforce outbound internet policies through forward proxies, blocking malicious sites, social media, or bandwidth-heavy services during work hours.

**Content Filtering**: Schools and enterprises filter inappropriate content, malware domains, and phishing sites at the proxy level rather than on individual devices.

**Bandwidth Optimization**: Forward proxies cache frequently accessed content (web pages, downloads), reducing external bandwidth usage and improving employee experience.

```bash
# Squid forward proxy configuration
acl allowed_sites dstdomain .company.com .partner-site.com
http_access allow allowed_sites
http_access deny all

# Client configuration
export http_proxy=http://proxy.company.com:3128
curl https://api.external-service.com/data
```

### Reverse Proxy (Inbound)

```
[External Clients] ──→ [Reverse Proxy] ──→ [Backend Server 1]
                                     ├──→ [Backend Server 2]
                                     └──→ [Backend Server 3]
```

The reverse proxy is the public face of your application. Key functions:

**TLS Termination**: Offloads TLS/SSL encryption and decryption from backend servers. The proxy handles certificate management, freeing application servers from costly crypto operations.

**Load Balancing**: Distributes requests across multiple backend servers using algorithms (round-robin, least connections, IP hash). This is how most internet-facing applications achieve horizontal scaling.

**Caching**: Caches static content (images, CSS, JS) and even dynamic responses at the edge, reducing load on backend servers and improving response times.

**Security**: Hides backend server IPs, protects against DDoS attacks (rate limiting, connection limits), provides web application firewall (WAF) capabilities, and prevents direct access to application servers.

**Compression**: Compresses responses (gzip, brotli) before sending to clients, reducing bandwidth usage and improving load times.

**Request Routing**: Routes requests to different backends based on URL path, hostname, headers, or other attributes:

```nginx
# Nginx reverse proxy configuration
server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate     /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # Route /api/* to API servers
    location /api/ {
        proxy_pass http://api-backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Route /static/* to CDN or static file servers
    location /static/ {
        proxy_pass http://static-backend;
        proxy_cache STATIC;
        proxy_cache_valid 200 1h;
    }

    # Route /admin/* to admin service
    location /admin/ {
        proxy_pass http://admin-backend;
        # Restrict to internal IPs
        allow 10.0.0.0/8;
        deny all;
    }
}
```

### API Gateway: The Modern Reverse Proxy
An **API Gateway** is essentially an advanced reverse proxy specialized for microservice APIs:

- **Authentication & Authorization**: Centralize JWT validation, API key verification, OAuth token introspection at the gateway rather than in each microservice
- **Rate Limiting**: Apply per-user, per-IP, or per-API-key rate limits at the entry point
- **Request/Response Transformation**: Modify headers, rewrite URLs, convert between protocols (HTTP to gRPC, REST to GraphQL)
- **API Versioning**: Route `/v1/*` and `/v2/*` to different backend versions simultaneously
- **Monitoring & Logging**: Capture metrics (latency, error rates, request counts) and generate access logs in one place vs. every service
- **Circuit Breaking**: Stop routing to failing backends automatically

### Proxy vs Service Mesh (Sidecar Proxy)
A **service mesh** (like Istio with Envoy sidecars) pushes reverse proxy functionality to every service instance:

- Each service has its own sidecar proxy (the "data plane")
- Instead of a central reverse proxy handling all ingress, every inter-service call goes through local sidecar proxies
- Benefits: fine-grained traffic control, observability, and security for east-west (service-to-service) traffic
- API Gateway handles north-south (client-to-service) traffic; Service Mesh handles east-west

### Transparent vs Non-Transparent Proxy

- **Transparent Proxy**: Client doesn't know the proxy exists. Traffic is intercepted at the network level (router, switch) and redirected. No client configuration needed.
- **Non-Transparent Proxy**: Client is explicitly configured to use the proxy (browser settings, HTTP_PROXY environment variable, SDK configuration). Client sends requests directly to the proxy.

## Layman's Explanation

### Forward Proxy: The Office Mail Room
Think of a corporate mail room. When employees (internal clients) want to send mail to external recipients (websites), they don't walk to the post office themselves. All outgoing mail goes through the mail room (forward proxy). The mail room can:

- Check that mail doesn't violate company policy (content filtering)
- Log who's sending what (auditing)
- Use the company return address instead of the employee's name (anonymity)
- Cache frequently requested catalogs so they don't need to be ordered again (caching)

Outside world sees only "ACME Corp, 123 Main St"—not "Bob from Accounting, Cubicle 4B."

### Reverse Proxy: The Hotel Front Desk
Imagine a luxury hotel. Guests (external clients) never go directly to hotel rooms (backend servers). They always interact with the front desk (reverse proxy). The front desk:

- Checks guests in/verifies identity (authentication)
- Directs guests to the right room (routing)
- Handles room keys (TLS certificates)
- Takes room service orders and delivers food (request/response handling)
- If one room has a plumbing issue, moves guests to another room (failover)
- Stores extra towels/concierge info that guests need frequently (caching)

Guests think the front desk IS the hotel. They never know which room does what, and the hotel can renovate rooms (deploy new servers) without guests noticing.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Security Architecture**: Reverse proxies create a security perimeter. Backend servers should never be directly accessible from the internet. This single decision prevents entire classes of attacks.
- **TLS Strategy**: Where do you terminate TLS? At the reverse proxy? Re-encrypt to backend? End-to-end TLS? Each choice has performance and security implications. PCI compliance may require end-to-end encryption.
- **Scalability Pattern**: Reverse proxies enable horizontal scaling without changing client-facing DNS. Add/remove backend servers behind a proxy without any client-side awareness.
- **Protocol Translation**: Modern architectures use gRPC internally but expose REST externally. The proxy (API Gateway) translates between them. Legacy HTTP/1.1 frontend with HTTP/2 or gRPC backend is a common pattern.
- **Observability**: Centralized proxy logging provides a single pane of glass for all incoming traffic: request patterns, error rates, latency percentiles, user agents. Without this, debugging distributed systems becomes a nightmare.

### Business Impact
- **CDN Cost Savings**: A reverse proxy with caching can reduce origin server load by 70-90% for cacheable content, directly reducing compute costs.
- **DDoS Protection**: Rate limiting and connection throttling at the proxy layer prevents volumetric attacks from reaching application servers, avoiding downtime during attacks.
- **Zero-Downtime Deployments**: Proxy health checks enable rolling updates, blue-green deployments, and canary releases. Backend servers can be added/removed without a single dropped request.
- **Compliance**: Forward proxies enforce data loss prevention (DLP) policies, preventing sensitive data exfiltration. Required for SOC2, HIPAA, PCI DSS, and FedRAMP.

## On-Premises Examples

### Nginx Reverse Proxy
The most widely used reverse proxy, powering over 30% of all websites:

```nginx
# /etc/nginx/nginx.conf
upstream backend {
    least_conn;
    server 10.0.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8080 weight=2 backup;
}

server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 30s;

        # Rate limiting
        limit_req zone=api_limit burst=20 nodelay;
    }
}

# Rate limit zone definition
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
```

### HAProxy
Layer 4 and Layer 7 proxy optimized for high concurrency:

```haproxy
# /etc/haproxy/haproxy.cfg
frontend https-in
    bind *:443 ssl crt /etc/ssl/certs/example.pem
    mode http
    
    acl is_api path_beg /api/
    acl is_admin path_beg /admin/
    
    use_backend api_servers if is_api
    use_backend admin_servers if is_admin
    default_backend web_servers

backend api_servers
    mode http
    balance roundrobin
    option httpchk GET /health
    server api1 10.0.1.10:8080 check inter 2s
    server api2 10.0.1.11:8080 check inter 2s

backend admin_servers
    mode http
    balance source  # IP hash for session stickiness
    server admin1 10.0.1.20:8080 check
```

### Squid Forward Proxy
```bash
# /etc/squid/squid.conf
http_port 3128

# Access control lists
acl internal_network src 10.0.0.0/8
acl allowed_ports port 80 443
acl blocked_sites dstdomain .facebook.com .instagram.com

http_access allow internal_network allowed_ports !blocked_sites
http_access deny all

# Caching
cache_dir ufs /var/spool/squid 10000 16 256
maximum_object_size 128 MB
```

## AWS Examples

### Application Load Balancer (ALB) - Layer 7 Reverse Proxy
```hcl
resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.bucket
    enabled = true
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.app.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Content-based routing listener rules
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}
```

### Network Load Balancer (NLB) - Layer 4 Reverse Proxy
For ultra-low latency, high-throughput TCP/UDP traffic:

```hcl
resource "aws_lb" "tcp" {
  name               = "tcp-nlb"
  load_balancer_type = "network"
  subnets            = aws_subnet.public[*].id
}
```

### Amazon API Gateway
Fully managed API Gateway (advanced reverse proxy with API management):

```yaml
# AWS SAM template
HttpApi:
  Type: AWS::Serverless::HttpApi
  Properties:
    StageName: prod
    CorsConfiguration:
      AllowOrigins:
        - https://example.com
      AllowMethods:
        - GET
        - POST
    Auth:
      Authorizers:
        OAuthAuthorizer:
          JwtConfiguration:
            issuer: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_xxx
            audience:
              - my-client-id
      DefaultAuthorizer: OAuthAuthorizer

# Route: ${HttpApi}.execute-api.us-east-1.amazonaws.com/orders -> Lambda function
```

### AWS CloudFront (CDN as Reverse Proxy)
CloudFront acts as a globally distributed reverse proxy:

```hcl
resource "aws_cloudfront_distribution" "cdn" {
  origin {
    domain_name = aws_lb.app.dns_name
    origin_id   = "ALB"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    target_origin_id       = "ALB"
    viewer_protocol_policy = "redirect-to-https"
    
    cached_methods    = ["GET", "HEAD"]
    allowed_methods   = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    
    forwarded_values {
      query_string = true
      cookies {
        forward = "all"
      }
    }
    min_ttl     = 0
    default_ttl = 3600
    max_ttl     = 86400
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.cdn.arn
    ssl_support_method  = "sni-only"
  }
}
```

## GCP Examples

### Cloud Load Balancing (Global, Anycast)
Google's global reverse proxy with single anycast IP:

```bash
# Global HTTPS load balancer (Layer 7)
gcloud compute url-maps create web-map \
  --default-service backend-service-web

gcloud compute url-maps add-path-matcher web-map \
  --path-matcher-name api-matcher \
  --default-service backend-service-api \
  --path-rules "/api/*=backend-service-api,/admin/*=backend-service-admin"

gcloud compute target-https-proxies create https-lb-proxy \
  --url-map=web-map \
  --ssl-certificates=example-cert

gcloud compute forwarding-rules create https-content-rule \
  --global \
  --target-https-proxy=https-lb-proxy \
  --ports=443
```

### Apigee API Gateway
Google's enterprise API management platform:

```xml
<!-- Apigee proxy configuration -->
<ProxyEndpoint name="default">
  <HTTPProxyConnection>
    <BasePath>/v1/orders</BasePath>
    <VirtualHost>default</VirtualHost>
  </HTTPProxyConnection>
  
  <RouteRule name="default">
    <TargetEndpoint>orders-backend</TargetEndpoint>
  </RouteRule>
  
  <FaultRules>
    <FaultRule name="rate-limit-exceeded">
      <Step>
        <Name>RaiseFault-429</Name>
      </Step>
    </FaultRule>
  </FaultRules>
</ProxyEndpoint>
```

### Identity-Aware Proxy (IAP)
Application-layer access control reverse proxy:

```bash
gcloud iap web enable --resource-type=backend-services \
  --oauth2-client-id=CLIENT_ID \
  --oauth2-client-secret=CLIENT_SECRET \
  backend-service-web
```

## Azure Examples

### Azure Application Gateway (Layer 7)
Regional reverse proxy with WAF, TLS termination, and cookie-based affinity:

```bash
az network application-gateway create \
  --name appGateway \
  --resource-group myResourceGroup \
  --location eastus \
  --sku WAF_v2 \
  --capacity 2 \
  --vnet-name myVNet \
  --subnet appGatewaySubnet \
  --http-settings-protocol Http \
  --public-ip-address appGatewayPublicIP
```

### Azure Front Door (Global Layer 7)
Microsoft's global anycast reverse proxy and CDN:

```json
{
  "frontendEndpoints": [
    {"hostName": "www.example.com", "sessionAffinityEnabledState": "Enabled"}
  ],
  "routingRules": [
    {
      "name": "api-route",
      "frontendEndpoints": [{"id": "www.example.com"}],
      "acceptedProtocols": ["Https"],
      "patternsToMatch": ["/api/*"],
      "forwardingProtocol": "HttpsOnly",
      "backendPool": {"id": "api-backend-pool"},
      "enabledState": "Enabled"
    }
  ],
  "backendPools": [
    {
      "name": "api-backend-pool",
      "backends": [
        {"address": "api-app.azurewebsites.net", "httpPort": 80, "httpsPort": 443}
      ],
      "healthProbeSettings": {
        "protocol": "Https", "path": "/health", "intervalInSeconds": 30
      }
    }
  ]
}
```

### Azure API Management (APIM)
Enterprise API gateway with developer portal:

```bash
az apim create \
  --name myAPIM \
  --resource-group myResourceGroup \
  --publisher-email admin@example.com \
  --publisher-name "Example Corp" \
  --sku-name Developer
```

```xml
<!-- APIM inbound policy: rate limiting, JWT validation, transformation -->
<policies>
  <inbound>
    <rate-limit calls="100" renewal-period="60" />
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
      <openid-config url="https://login.microsoftonline.com/tenant/.well-known/openid-configuration" />
    </validate-jwt>
    <set-backend-service base-url="https://api-backend.azurewebsites.net" />
  </inbound>
</policies>
```

## Summary Decision Matrix

| Requirement | Forward Proxy | Reverse Proxy | API Gateway |
|------------|---------------|---------------|-------------|
| Primary direction | Outbound (internal → internet) | Inbound (internet → internal) | Inbound (internet → APIs) |
| Key use case | Access control, anonymity | Load balancing, TLS termination | API management, auth, rate limiting |
| Client awareness | Explicit (proxy settings) or transparent | None (client thinks it's the server) | None (API consumer sees gateway URL) |
| Common tools | Squid, Tinyproxy | Nginx, HAProxy, Envoy | Kong, Apigee, AWS API Gateway |
| OSI layer | L4 or L7 | L4 or L7 | L7 only |

Proxies are the architectural glue that connects clients to servers securely, scalably, and observably. Forward proxies control what leaves your network; reverse proxies control what enters your application. API gateways extend the reverse proxy pattern with API-specific lifecycle management. Together, they form the traffic management backbone of every production system.
