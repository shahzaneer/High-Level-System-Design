# AWS Load Balancer Examples

---

## 1. Application Load Balancer (ALB)

- **Layer:** L7 (HTTP/HTTPS, HTTP/2, WebSocket, gRPC)
- **Algorithms:**

  - Round robin (default)
  - Least outstanding requests (similar to least connections, but for HTTP requests)

**Key Features:**

- Path-based routing: Different URLs to different target groups
- Host-based routing: Multiple domains on one load balancer
- HTTP header routing: Route based on any HTTP header
- Query string routing: Route based on URL parameters
- Request/response header manipulation: Add/remove/modify headers
- SSL/TLS termination: Supports SNI (multiple certificates)
- WAF integration: AWS WAF for application security
- Authentication: Built-in OIDC/SAML authentication
- Lambda targets: Can invoke Lambda functions directly
- Sticky sessions: Cookie-based (application or ALB-generated)

**Configuration Example (Terraform):**

```hcl
resource "aws_lb" "main" {
  name               = "my-alb"
  load_balancer_type = "application"
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_b.id]
  security_groups    = [aws_security_group.alb.id]
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.cert.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# Path-based routing
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn

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

# Header-based routing (for canary deployment)
resource "aws_lb_listener_rule" "canary" {
  listener_arn = aws_lb_listener.https.arn

  action {
    type = "forward"
    forward {
      target_group {
        arn    = aws_lb_target_group.prod.arn
        weight = 90
      }
      target_group {
        arn    = aws_lb_target_group.canary.arn
        weight = 10
      }
    }
  }

  condition {
    path_pattern {
      values = ["/*"]
    }
  }
}

resource "aws_lb_target_group" "web" {
  name     = "web-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path                = "/health"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200"
  }

  stickiness {
    type            = "lb_cookie"
    enabled         = true
    cookie_duration = 86400  # 24 hours
  }
}
```

**When to Use:**

- Modern web applications
- Microservices architectures
- Need L7 features (path/host routing, authentication)
- API gateways (though AWS API Gateway is more feature-rich for APIs)
- WebSocket applications

**Real-World Scenario:**

- E-commerce application:

  - `/` → Web server target group (t3.medium instances)
  - `/api/*` → API server target group (c5.large instances)
  - `/images/*` → Image processing target group (g4dn instances with GPUs)
  - Users with `X-Beta-User: true` header → Canary target group (new version)

---

## 2. Network Load Balancer (NLB)

- **Layer:** L4 (TCP, UDP, TLS)
- **Algorithms:** Flow hash algorithm (based on protocol, source IP, source port, destination IP, destination port)

  - Selects target using hash, maintains connection to that target

**Key Features:**

- Ultra-low latency: Microsecond latencies
- Extreme performance: Millions of requests per second, 100s of Gbps
- Static IP: Supports Elastic IPs (useful for whitelisting)
- Preserve source IP: Clients' IP addresses preserved
- TLS termination: Can terminate TLS while staying L4 for other traffic
- Cross-zone load balancing: Optional (disabled by default for NLB)
- Connection draining: Gracefully handle instance removal
- PrivateLink support: Expose services via VPC endpoints

**Configuration Example:**

```hcl
resource "aws_lb" "network" {
  name               = "my-nlb"
  load_balancer_type = "network"
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_b.id]

  enable_cross_zone_load_balancing = true

  # Assign static Elastic IPs
  subnet_mapping {
    subnet_id     = aws_subnet.public_a.id
    allocation_id = aws_eip.lb_a.id
  }

  subnet_mapping {
    subnet_id     = aws_subnet.public_b.id
    allocation_id = aws_eip.lb_b.id
  }
}

resource "aws_lb_listener" "tcp" {
  load_balancer_arn = aws_lb.network.arn
  port              = 3306  # MySQL
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.database.arn
  }
}

resource "aws_lb_target_group" "database" {
  name     = "db-tg"
  port     = 3306
  protocol = "TCP"
  vpc_id   = aws_vpc.main.id

  health_check {
    protocol            = "TCP"
    port                = 3306
    interval            = 30
    healthy_threshold   = 3
    unhealthy_threshold = 3
  }

  deregistration_delay = 300  # Connection draining
  preserve_client_ip   = true  # Preserve source IP
}
```

**When to Use:**

- Non-HTTP protocols (databases, gaming servers, IoT, custom TCP/UDP)
- Need extreme performance (millions of requests/second)
- Require static IP addresses
- Source IP preservation critical
- TLS passthrough needed (end-to-end encryption)

**Real-World Scenario:**

- **Database Read Replicas:** Load balance read traffic across RDS read replicas
- **Gaming Server:** Multiplayer game with custom UDP protocol; preserve player IPs
- **IoT:** Millions of IoT devices connecting via MQTT (TCP port 1883); extreme connection counts

---

## 3. Gateway Load Balancer (GWLB)

- **Layer:** L3 (IP packets)
- **What It Is:** Specialized load balancer for deploying, scaling, and managing third-party virtual appliances (firewalls, IDS/IPS, DPI)
- **How It Works:** Uses GENEVE protocol to encapsulate traffic, send through appliances, then return to GWLB. Transparent to applications

**When to Use:**

- Centralized security inspection (all traffic through security appliances)
- IDS/IPS deployment
- Third-party firewalls (Palo Alto, Fortinet, Check Point)
- Network traffic analysis

**Real-World Scenario:**

- Enterprise needs all internet-bound traffic inspected by Palo Alto firewall cluster
- GWLB distributes traffic across firewall instances and scales automatically

---

## 4. Classic Load Balancer (CLB) (Legacy - not recommended)

- **Layer:** L4 and basic L7
- **Status:** Previous generation, being phased out. Use ALB or NLB instead
