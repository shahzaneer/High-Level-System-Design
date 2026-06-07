# Web Application Firewall (WAF) & OWASP

## Introduction
Web Application Firewalls are the specialized security layer that protects web applications from the most common and dangerous attacks. Unlike network firewalls that operate at Layer 3/4 (IP, ports), WAFs operate at Layer 7 (HTTP/HTTPS) and understand the application protocol deeply enough to detect and block malicious requests that look like normal traffic to a network firewall.

The OWASP (Open Web Application Security Project) Top 10 is the industry-standard taxonomy of the most critical web application security risks. Updated every few years, it serves as both an awareness document and a compliance baseline. Nearly every security standard (PCI DSS Requirement 6.5, SOC 2 CC6.1) explicitly references OWASP Top 10 protections. A solution architect must understand both the WAF's capabilities and the attack types it defends against.

## Definition
A **Web Application Firewall (WAF)** is a Layer 7 firewall that monitors, filters, and blocks HTTP/HTTPS traffic to and from a web application. It inspects every aspect of the request—headers, body, query parameters, cookies, and URL—against a set of rules designed to detect common attack patterns.

The **OWASP Top 10** is a consensus document listing the ten most critical security risks to web applications, maintained by the OWASP Foundation. The current list (2021 edition) includes:

1. **Broken Access Control**: Users acting outside of intended permissions
2. **Cryptographic Failures**: Sensitive data exposure due to weak/nonexistent encryption
3. **Injection**: SQL, NoSQL, OS command, LDAP injection via untrusted input
4. **Insecure Design**: Missing or ineffective security controls in architecture
5. **Security Misconfiguration**: Default accounts, verbose errors, unnecessary features
6. **Vulnerable & Outdated Components**: Unpatched libraries, frameworks, OS
7. **Identification & Authentication Failures**: Weak passwords, missing MFA, session fixation
8. **Software & Data Integrity Failures**: Unsafe deserialization, CI/CD pipeline compromise
9. **Security Logging & Monitoring Failures**: Undetected breaches due to missing logs/alerts
10. **Server-Side Request Forgery (SSRF)**: Server fetching attacker-controlled URLs

## Concept Explanation

### WAF Rule Types

#### Allow Rules (Positive Security / Allowlist)
Only requests matching expected patterns are allowed. Most secure but requires deep application knowledge.

```
Rule: Allow GET requests to /api/orders/* with valid JWT in Authorization header
      and specific query parameters (page, limit, status)
      
All other requests → Blocked
```

#### Deny Rules (Negative Security / Blocklist)
Known attack patterns are blocked. Easier to start but can be bypassed by novel attacks.

```
Rule: Block requests containing:
      - ' OR 1=1 -- (SQL injection)
      - <script>alert(1)</script> (XSS)
      - ../../../etc/passwd (Path traversal)
      - $(cat /etc/passwd) (Command injection)
```

#### Rate-Based Rules
Block sources that exceed request thresholds (volumetric protection).

#### Anomaly Scoring
Each rule has a score. When cumulative score exceeds threshold, request is blocked.

### Core WAF Protections

#### SQL Injection Protection
```python
# WITHOUT WAF: vulnerable SQL query
@app.route('/search')
def search():
    query = request.args.get('q', '')
    # DANGEROUS: string concatenation
    sql = f"SELECT * FROM products WHERE name LIKE '%{query}%'"
    results = db.execute(sql)

# WITH WAF: WAF inspects query parameter
# WAF sees: q=foo%27%20OR%201%3D1%20--
# Decoded: foo' OR 1=1 --
# WAF rule: block SQL metacharacters (', ;, --, /*) in query params
# Request blocked → 403 Forbidden

# STILL IMPORTANT: Use parameterized queries
@app.route('/search')
def search():
    query = request.args.get('q', '')
    sql = "SELECT * FROM products WHERE name LIKE %s"
    results = db.execute(sql, [f'%{query}%'])  # Parameterized
```

#### Cross-Site Scripting (XSS) Protection
```python
# WAF inspects both request body and response body

# Block XSS in request:
# POST /comments body: comment=<script>alert('xss')</script>
# WAF rule detects <script> tag in POST body → blocked

# WAF can also add Content-Security-Policy header to responses
# Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-random123'
```

#### Path Traversal Protection
```python
# Attack: GET /download?file=../../../etc/passwd
# WAF rule: block ../ and encoded variants (%2e%2e%2f, ..%252f, ..%c0%af) in URL
```

#### Bot Management
```python
# WAF identifies bots through:
# - Known bot signatures (User-Agent patterns)
# - Behavior analysis (request rate, navigation patterns)
# - Browser fingerprinting (JavaScript challenge)
# - CAPTCHA integration

# WAF response to suspected bot:
# HTTP 200: <html><script>/* JavaScript challenge */</script></html>
# Legitimate browsers execute JS → get cookie → retry → allowed
# Bots that can't execute JS → blocked silently
```

### WAF Deployment Modes

**Blocking Mode**: Malicious requests are immediately rejected with 403/406. Use in production after tuning false positives.

**Detection/Logging Mode**: Rules are evaluated and logged but requests are not blocked. Use during initial deployment to baseline traffic and identify false positives.

```
WAF in Detection Mode:
  Request → WAF inspects → matches rule "XSS attack" → logs match → passes request to origin
  Security team reviews logs, confirms legitimate traffic patterns, adds exceptions

WAF in Blocking Mode:
  Request → WAF inspects → matches rule "XSS attack" → blocks → 403 + logged
```

### OWASP Top 10 in Practice

#### A01:2021 Broken Access Control
```python
# Vulnerable: no authorization check
@app.route('/api/orders/<order_id>')
def get_order(order_id):
    return Order.query.get(order_id)
# Alice can view Bob's orders by changing order_id in URL

# Secure: with WAF + application-level check
@app.route('/api/orders/<order_id>')
@require_auth
def get_order(order_id):
    order = Order.query.get(order_id)
    if order.customer_id != current_user.id:
        abort(403)
    return order.to_dict()
# WAF adds layer: can enforce that /api/orders/* requires valid JWT
```

#### A03:2021 Injection
```python
# BAD: OS command injection
@app.route('/ping')
def ping():
    host = request.args.get('host', '')
    # Attacker sends: host=8.8.8.8; cat /etc/passwd
    output = subprocess.check_output(f"ping -c 1 {host}", shell=True)
    return output

# WAF blocks: semicolons, pipes, backticks, $(, && in query params

# GOOD: Use subprocess with list (no shell), validate input
import ipaddress
ip = ipaddress.ip_address(host)  # validates it's actually an IP
output = subprocess.check_output(["ping", "-c", "1", str(ip)])
```

#### A10:2021 SSRF (Server-Side Request Forgery)
```python
# Vulnerable: server fetches user-supplied URL
@app.route('/fetch')
def fetch_url():
    url = request.args.get('url')
    r = requests.get(url)  # Attacker: url=http://169.254.169.254/latest/meta-data/
    return r.text
# Attacker can access cloud metadata service, internal services

# WAF blocks: requests to internal IPs (127.0.0.1, 10.x, 172.16-31.x, 192.168.x, 169.254.x)

# Application-level fix: URL allowlist + parse and validate
from urllib.parse import urlparse
parsed = urlparse(url)
if parsed.hostname not in ['api.public.com', 'cdn.example.com']:
    abort(400)
```

### WAF Bypass Techniques (Defense in Depth)

Attackers attempt to bypass WAF rules:
```
Original:     <script>alert(1)</script>
Obfuscated:   <scr<script>ipt>alert(1)</scr</script>ipt>
Encoded:      %3Cscript%3Ealert(1)%3C%2Fscript%3E
Unicode:      \u003cscript\u003ealert(1)\u003c/script\u003e
Multibyte:    <scrİpt>alert(1)</scrİpt>
```

This is why WAF alone is insufficient. Application code must still validate and sanitize inputs. Defense in depth: WAF + input validation + output encoding + CSP headers.

## Layman's Explanation

### WAF: The Metal Detector at the Stadium Entrance
A stadium (your web application) has thousands of visitors (HTTP requests) every hour. Most are legitimate fans coming to watch the game. Some might be carrying weapons (malicious payloads).

The metal detector (WAF) at the entrance:
- Scans every visitor for known weapons (SQL injection, XSS patterns)
- If you beep (match a rule), security pulls you aside for additional screening
- Known threats are immediately turned away (403 forbidden)
- The metal detector knows the difference between car keys (legitimate JSON) and a knife (SQL metacharacters in query params)

But the metal detector alone isn't enough:
- A weapon might be made of ceramic (novel attack the detector doesn't recognize)
- The stadium also needs locked doors (authentication), restricted areas (authorization), security cameras (logging), and staff training (developer security awareness)

### OWASP Top 10: The FBI's Most Wanted List
Just like the FBI maintains a list of the most dangerous fugitives, OWASP maintains a list of the top 10 most dangerous web vulnerabilities. Every developer and architect should know this list. When you're designing an application, you go down the list and ask: "Are we protected against #1? #2? #3?"

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **WAF Placement**: At the CDN edge (CloudFront, Cloudflare), at the load balancer (ALB, Application Gateway), or at the API gateway? Edge placement stops attacks furthest from origin. Multiple WAF layers (defense in depth) is best practice.
- **Rule Tuning vs False Positives**: Overly aggressive WAF rules block legitimate users (false positives), causing lost revenue. Too lenient rules miss attacks (false negatives). The tuning process requires analyzing traffic for weeks, creating custom exceptions for legitimate-but-unusual patterns, then moving to blocking mode.
- **Managed vs Custom Rules**: Managed rule sets (AWS Managed Rules, OWASP ModSecurity Core Rule Set) provide comprehensive baseline protection updated for new CVEs. Custom rules address application-specific logic (e.g., "block requests with `admin=true` in cookie unless from internal IP").
- **WAF + API Security**: REST/GraphQL APIs have different attack surfaces than web pages. WAF needs API-specific rules: schema validation, method enforcement (only GET/POST allowed), content-type checking, and response inspection for data leakage.

### Business Impact
- **Compliance**: PCI DSS Requirement 6.6 mandates either a WAF or code review for all public-facing web applications. A WAF is often the practical path to compliance for organizations with many applications.
- **Breach Prevention**: The average cost of a data breach is $4.45 million (IBM 2023). WAFs prevent the most common attack vectors (SQL injection, XSS) that account for a significant percentage of breaches.
- **Time-to-Patch Buffer**: When a critical CVE is announced (e.g., Log4Shell, Spring4Shell), organizations need days or weeks to patch all systems. A WAF can deploy a virtual patch rule in minutes, blocking exploitation attempts while the actual fix is rolled out.
- **Bot Protection**: Bots account for 40-50% of internet traffic. WAF bot management protects against credential stuffing, inventory hoarding, price scraping, and DDoS—all of which directly impact revenue.

## On-Premises Examples

### ModSecurity + OWASP Core Rule Set
The open-source WAF engine, commonly deployed with Nginx or Apache:

```bash
# Install ModSecurity with Nginx
apt-get install libmodsecurity3 nginx-module-modsecurity

# Enable ModSecurity in nginx.conf
load_module modules/ngx_http_modsecurity_module.so;

server {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
    
    location / {
        proxy_pass http://backend;
        ModSecurityEnabled on;
    }
}
```

```bash
# modsecurity.conf
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess On
SecAuditEngine RelevantOnly
SecAuditLog /var/log/modsec_audit.log

# Include OWASP Core Rule Set
Include /etc/nginx/modsec/crs-setup.conf
Include /etc/nginx/modsec/rules/*.conf

# Custom rule: block requests with suspicious User-Agent
SecRule REQUEST_HEADERS:User-Agent "^(sqlmap|nikto|nessus|nmap)" \
  "id:1000, phase:1, deny, status:403, msg:'Security scanner detected'"

# Custom rule: block XML external entity (XXE) in XML body
SecRule REQUEST_BODY "<!ENTITY\s+[^\s]+\s+SYSTEM" \
  "id:1001, phase:2, deny, status:403, msg:'XXE attack detected'"
```

### NAXSI (WAF for Nginx)
Whitelist-based (positive security) WAF:

```nginx
location / {
    # Enable NAXSI learning mode (logs without blocking)
    include /etc/nginx/naxsi.rules;
    
    proxy_pass http://backend;
}

location /RequestDenied {
    return 418;  # Teapot
}
```

```naxsi
# naxsi.rules
LearningMode;
SecRulesEnabled;
DeniedUrl "/RequestDenied";

CheckRule "$SQL >= 8" BLOCK;
CheckRule "$RFI >= 8" BLOCK;
CheckRule "$TRAVERSAL >= 4" BLOCK;
CheckRule "$XSS >= 8" BLOCK;
CheckRule "$EVADE >= 4" BLOCK;
```

### OWASP ZAP (Penetration Testing)
```bash
# Automated security scan with ZAP
zap-cli quick-scan --self-contained \
  --start-options="-config api.disablekey=true" \
  http://localhost:8080

# Full active scan
zap-cli open-url http://localhost:8080
zap-cli spider http://localhost:8080
zap-cli active-scan http://localhost:8080
zap-cli report -o security-report.html -f html
```

## AWS Examples

### AWS WAF
```hcl
resource "aws_wafv2_web_acl" "app" {
  name        = "app-waf"
  description = "WAF for application"
  scope       = "REGIONAL"  # or CLOUDFRONT for global

  default_action {
    allow {}
  }

  # Allow rule: only specific countries
  rule {
    name     = "geo-restrict"
    priority = 0

    action {
      allow {}
    }

    statement {
      geo_match_statement {
        country_codes = ["US", "CA", "GB", "DE", "FR", "JP", "AU"]
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "GeoRestrictRule"
      sampled_requests_enabled   = true
    }
  }

  # AWS Managed Rules: OWASP Top 10 protections
  rule {
    name     = "managed-owasp"
    priority = 1

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
        
        rule_action_override {
          name = "SizeRestrictions_BODY"
          action_to_use {
            count {}  # only log, don't block
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "ManagedOWASP"
      sampled_requests_enabled   = true
    }
  }

  # SQL Injection protection
  rule {
    name     = "managed-sqli"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLiProtection"
      sampled_requests_enabled   = true
    }
  }

  # Rate limiting
  rule {
    name     = "rate-limit"
    priority = 3

    action {
      block {
        custom_response {
          response_code = 429
          response_header {
            name  = "Retry-After"
            value = "60"
          }
        }
      }
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }

  # Custom rule: block requests to /admin from non-internal IPs
  rule {
    name     = "admin-access"
    priority = 4

    action {
      block {}
    }

    statement {
      and_statement {
        statement {
          byte_match_statement {
            search_string = "/admin/"
            field_to_match {
              uri_path {}
            }
            text_transformation {
              priority = 0
              type     = "NONE"
            }
            positional_constraint = "STARTS_WITH"
          }
        }
        statement {
          not_statement {
            statement {
              ip_set_reference_statement {
                arn = aws_wafv2_ip_set.admin_ips.arn
              }
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AdminAccess"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "AppWAF"
    sampled_requests_enabled   = true
  }
}

# IP set for admin access
resource "aws_wafv2_ip_set" "admin_ips" {
  name               = "admin-ips"
  scope              = "REGIONAL"
  ip_address_version = "IPV4"
  addresses          = ["10.0.0.0/8", "203.0.113.0/24"]
}
```

### AWS WAF Log Analysis with Athena
```sql
-- Query WAF logs in S3 via Athena
CREATE EXTERNAL TABLE waf_logs (
  `timestamp` bigint,
  `formatVersion` int,
  `webaclId` string,
  `terminatingRuleId` string,
  `terminatingRuleType` string,
  `action` string,
  `httpSourceName` string,
  `httpSourceId` string,
  `ruleGroupList` array<struct<...>>,
  `rateBasedRuleList` array<struct<...>>,
  `nonTerminatingMatchingRules` array<struct<...>>,
  `httpRequest` struct<
    clientIp: string,
    country: string,
    headers: array<struct<name: string, value: string>>,
    uri: string,
    args: string,
    httpVersion: string,
    httpMethod: string,
    requestId: string
  >
)
PARTITIONED BY (dt string)
STORED AS PARQUET
LOCATION 's3://waf-logs-bucket/AWSLogs/';

-- Top blocked IPs
SELECT httpRequest.clientIp, COUNT(*) as block_count
FROM waf_logs
WHERE action = 'BLOCK' AND dt = '2024-01-01'
GROUP BY httpRequest.clientIp
ORDER BY block_count DESC
LIMIT 10;
```

### AWS Shield Advanced (DDoS + WAF Integration)
```hcl
resource "aws_shield_protection" "app" {
  name         = "app-protection"
  resource_arn = aws_lb.app.arn

  application_layer_automatic_response_configuration {
    action {
      block {}
    }
    status = "ENABLED"
  }
}
```

## GCP Examples

### Cloud Armor (WAF)
```bash
# Create security policy
gcloud compute security-policies create app-waf-policy \
  --description "WAF policy for application"

# Default rule: allow
gcloud compute security-policies rules update 2147483647 \
  --security-policy=app-waf-policy \
  --action=allow \
  --description="Default allow"

# Pre-configured WAF rules (OWASP Top 10 equivalent)
gcloud compute security-policies rules create 1000 \
  --security-policy=app-waf-policy \
  --expression="evaluatePreconfiguredWaf('sqli-v33-stable', {'sensitivity': 2})" \
  --action=deny-403

gcloud compute security-policies rules create 1010 \
  --security-policy=app-waf-policy \
  --expression="evaluatePreconfiguredWaf('xss-v33-stable', {'sensitivity': 2})" \
  --action=deny-403

gcloud compute security-policies rules create 1020 \
  --security-policy=app-waf-policy \
  --expression="evaluatePreconfiguredWaf('rfi-v33-stable', {'sensitivity': 2})" \
  --action=deny-403

gcloud compute security-policies rules create 1030 \
  --security-policy=app-waf-policy \
  --expression="evaluatePreconfiguredWaf('lfi-v33-stable', {'sensitivity': 2})" \
  --action=deny-403

gcloud compute security-policies rules create 1040 \
  --security-policy=app-waf-policy \
  --expression="evaluatePreconfiguredWaf('rce-v33-stable', {'sensitivity': 2})" \
  --action=deny-403

# Rate limiting
gcloud compute security-policies rules create 2000 \
  --security-policy=app-waf-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=300 \
  --conform-action=allow \
  --exceed-action=deny-429

# Geo restriction
gcloud compute security-policies rules create 3000 \
  --security-policy=app-waf-policy \
  --expression="origin.region_code == 'US' || origin.region_code == 'CA'" \
  --action=allow

# Attach to backend service
gcloud compute backend-services update my-backend \
  --security-policy=app-waf-policy
```

### Cloud Armor Custom Rules
```bash
# Custom rule language (CEL - Common Expression Language)
# Block requests with suspicious headers
gcloud compute security-policies rules create 1050 \
  --security-policy=app-waf-policy \
  --expression="request.headers['user-agent'].contains('sqlmap') || request.headers['user-agent'].contains('nikto')" \
  --action=deny-403

# Block requests to admin paths from non-internal IPs
gcloud compute security-policies rules create 1060 \
  --security-policy=app-waf-policy \
  --expression="request.path.matches('/admin(/.*)?') && !inIpRange(origin.ip, '10.0.0.0/8')" \
  --action=deny-403
```

## Azure Examples

### Azure Application Gateway WAF v2
```bash
az network application-gateway waf-policy create \
  --name app-waf-policy \
  --resource-group myResourceGroup \
  --location eastus \
  --type OWASP \
  --version 3.2

# Enable managed rule sets
az network application-gateway waf-policy managed-rule-set add \
  --policy-name app-waf-policy \
  --resource-group myResourceGroup \
  --type OWASP \
  --version 3.2 \
  --group-name "REQUEST-942-APPLICATION-ATTACK-SQLI" \
  --rules "942100 942110 942120"

# Custom rule: rate limit by IP
az network application-gateway waf-policy custom-rule create \
  --policy-name app-waf-policy \
  --resource-group myResourceGroup \
  --name RateLimitRule \
  --priority 10 \
  --action Block \
  --rule-type RateLimitRule \
  --rate-limit-threshold 100 \
  --rate-limit-duration OneMin \
  --match-conditions condition="RemoteAddr=*"

# Custom rule: geo restriction
az network application-gateway waf-policy custom-rule create \
  --policy-name app-waf-policy \
  --resource-group myResourceGroup \
  --name GeoRestrict \
  --priority 20 \
  --action Block \
  --rule-type MatchRule \
  --match-conditions '[{"matchVariable":"RemoteAddr","operator":"GeoMatch","matchValues":["ZZ"],"negationConditon":false,"transforms":[]}]'
```

### Azure Front Door WAF
```json
{
  "name": "app-frontdoor-waf",
  "properties": {
    "policySettings": {
      "enabledState": "Enabled",
      "mode": "Prevention",
      "requestBodyCheck": true,
      "maxRequestBodySizeInKb": 128
    },
    "managedRules": {
      "managedRuleSets": [
        {
          "ruleSetType": "Microsoft_DefaultRuleSet",
          "ruleSetVersion": "2.1",
          "ruleGroupOverrides": [
            {
              "ruleGroupName": "SQLI",
              "rules": [
                {
                  "ruleId": "942110",
                  "action": "Block",
                  "enabledState": "Enabled"
                }
              ]
            }
          ]
        },
        {
          "ruleSetType": "Microsoft_BotManagerRuleSet",
          "ruleSetVersion": "1.0"
        }
      ]
    },
    "customRules": {
      "rules": [
        {
          "name": "RateLimit",
          "enabledState": "Enabled",
          "priority": 1,
          "ruleType": "RateLimitRule",
          "rateLimitDurationInMinutes": 1,
          "rateLimitThreshold": 100,
          "matchConditions": [
            {
              "matchVariable": "RemoteAddr",
              "operator": "IPMatch",
              "matchValue": ["*"]
            }
          ],
          "action": "Block"
        }
      ]
    }
  }
}
```

## Summary Decision Matrix

| WAF Capability | AWS WAF | Cloud Armor | Azure WAF |
|---------------|---------|-------------|-----------|
| Managed rule sets | AWS Managed Rules + Marketplace | Pre-configured WAF rules | Microsoft Default Rule Set + OWASP CRS |
| Rate limiting | Yes (per-IP, 5-min window) | Yes (with ban duration) | Yes (per-IP, custom duration) |
| Bot protection | AWS Managed Rules Bot Control | reCAPTCHA integration | Microsoft Bot Manager |
| Custom rules | JSON-based, 1500 WCUs | CEL expression language | Custom JSON rules |
| Geo blocking | Yes (country codes) | Yes (region codes) | Yes (GeoMatch) |
| Logging destination | S3 + Kinesis + CloudWatch | Cloud Logging + BigQuery | Log Analytics + Event Hub |
| API-aware rules | Yes (body inspection, JSON parsing) | Yes (CEL on request fields) | Yes (MatchVariable on body, headers, cookies) |

A WAF is not optional for any internet-facing application. It provides a first line of defense against the most common and damaging web attacks, buys time during vulnerability disclosure cycles (virtual patching), and satisfies compliance requirements across PCI DSS, SOC 2, and HIPAA. However, a WAF is defense in depth—not a substitute for secure coding practices, input validation, parameterized queries, and output encoding at the application level. The architect's job is to ensure the WAF is properly tuned, continuously monitored, and never trusted as the sole security control.
