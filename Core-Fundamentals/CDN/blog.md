# Content Delivery Networks (CDN)

## Introduction
Content Delivery Networks (CDNs) are globally distributed networks of proxy servers that cache content closer to end users, dramatically reducing latency and origin server load. The CDN concept was pioneered by Akamai in 1998 to solve the "World Wide Wait" problem—users in Asia waiting 10+ seconds for pages hosted in North America. Today, CDNs deliver the vast majority of internet content: streaming video (Netflix), social media images (Facebook), software downloads, API responses, and entire websites.

Modern CDNs have evolved from simple static asset caches to programmable edge computing platforms capable of running application logic at the network edge. Services like Cloudflare Workers, AWS Lambda@Edge, and Fastly Compute@Edge allow developers to customize request/response handling, implement authentication, and run A/B tests before traffic ever reaches the origin server.

## Definition
A **Content Delivery Network (CDN)** is a geographically distributed group of servers (Points of Presence, or PoPs) that work together to deliver internet content quickly. The CDN caches copies of content (static files, API responses, video segments) at edge locations and serves user requests from the nearest PoP. This minimizes the physical distance data must travel, reducing round-trip time from hundreds of milliseconds to single digits.

Key metrics:
- **Edge locations (PoPs)**: Physical data centers where CDN servers are deployed (100-300+ globally for major providers)
- **Cache hit ratio**: Percentage of requests served from edge cache vs. forwarded to origin
- **Time to First Byte (TTFB)**: Time from request to first byte of response, a key performance indicator improved by CDNs

## Concept Explanation

### How a CDN Works

```
User in Tokyo ──→ [Tokyo Edge (PoP)]
                      │ Has cached version? ── Yes → Serve immediately (< 10ms)
                      │ No → Forward to origin
                      │         │
                      └─────────┴──→ [Origin Server in Virginia] (200ms+)
```

1. **DNS Resolution**: User requests `www.example.com`. CDN-aware DNS (Route 53, Cloudflare DNS) returns the IP of the nearest edge server based on the user's geographic location (GeoDNS, anycast routing).
2. **Edge Cache Check**: Request arrives at the nearest PoP. The CDN checks its cache:
   - **Cache HIT**: Content is present and not expired. Served immediately from edge.
   - **Cache MISS**: Content not cached or expired. Edge fetches from origin server, caches the response, and serves to user. Subsequent requests get cache HITs.
3. **Cache Invalidation**: When origin content changes, cached copies must be purged (invalidation) or wait for TTL to expire.

### Cache TTL and Freshness

Content freshness is controlled by HTTP caching headers:

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=3600, s-maxage=86400
# max-age: browser cache duration (1 hour)
# s-maxage: shared cache/CDN cache duration (24 hours)

ETag: "abc123"                           # version identifier
Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT
```

**Cache invalidation patterns**:
- **Time-based (TTL)**: CDN revalidates after the specified duration. Use conditional requests (`If-None-Match`, `If-Modified-Since`) to avoid re-downloading unchanged content.
- **Purge/Invalidation**: CDN API call to immediately remove cached content. Used for urgent updates (security patches, pricing changes).
- **Versioned URLs / Cache Busting**: Append version hash to filenames (`app.abc123.js`). Every deploy creates new URLs, never needing invalidation.

```python
# Flask example: versioned static files
@app.route('/static/<version>/<path:filename>')
def static_files(version, filename):
    response = send_from_directory('static', filename)
    response.headers['Cache-Control'] = 'public, max-age=31536000, immutable'
    # immutable: tells CDN/browser this content will never change for this URL
    return response
```

### CDN Architectures

#### Pull CDN (Origin Pull)
The CDN automatically fetches content from the origin server when it receives its first request for that content. Simplest setup—no explicit upload needed.

```
Origin: http://origin.example.com/images/photo.jpg
CDN URL: https://cdn.example.com/images/photo.jpg
First request → CDN fetches from origin → caches → serves
Subsequent requests → CDN serves from cache
```

Best for: Websites, web apps, content that changes occasionally.

#### Push CDN
Content is explicitly uploaded/pushed to the CDN in advance (often via API or upload tool). The CDN never contacts the origin server.

```bash
# Push content to CDN
aws s3 cp video.mp4 s3://my-bucket/
# CloudFront pulls from S3 or content can be pre-warmed
```

Best for: Large files, video streaming, software downloads where first-request latency matters, and for guaranteed availability independent of origin.

#### Hybrid CDN
Combines push for known popular content with pull for long-tail content. Netflix pushes popular new releases to edges and uses pull for niche catalog items.

### Dynamic Content Acceleration
CDNs optimize not just static content but also dynamic API responses:

- **Route optimization**: CDN knows the fastest network path between user and origin (bypasses congested internet peering points)
- **Connection multiplexing**: CDN maintains persistent, warmed connections to origin, avoiding TCP/TLS handshake latency
- **Protocol optimization**: HTTP/2, HTTP/3 (QUIC) between user and CDN edge, optimizing the "last mile"
- **Edge computing**: Run code at the edge to personalize content without origin round-trip

### Security Features

Modern CDNs provide critical security services:

- **DDoS Protection**: Absorb volumetric attacks at edge—hundreds of PoPs scatter the attack surface, making it impossible to overwhelm any single point
- **Web Application Firewall (WAF)**: Filter malicious requests (SQL injection, XSS) at the edge
- **Bot Management**: Detect and block automated traffic, credential stuffing, and scraping
- **TLS Termination**: Handle TLS at the edge with globally distributed certificates
- **Origin Shield**: Additional caching layer between edge PoPs and origin, reducing the number of requests that reach origin servers during cache misses

## Layman's Explanation

### The Library Network Analogy
Imagine a city with a main central library (origin server) storing all books (website content). Without a CDN:
- Every resident must travel to the central library for every book, even if they live on the outskirts (50-minute trip each way)

With a CDN (neighborhood branch libraries):
- The library system opens small branches in every neighborhood (edge locations/PoPs)
- Each branch stocks the most popular books (cached content)
- When you need a book that's at your branch, you walk 2 minutes
- When you need a rare book (cache miss), the branch fetches it from the central library and also keeps a copy for the next person
- If a new edition comes out (content update), the branch replaces old copies with new ones (invalidation)
- During a city-wide book club event (traffic spike), branches absorb the load so the main library doesn't collapse

### Netflix's Open Connect
Netflix's CDN strategy: Instead of paying Akamai or CloudFront, Netflix built their own CDN (Open Connect) and placed appliances inside ISPs worldwide. During peak hours, 95% of Netflix's traffic is served from ISP-local Open Connect appliances, never touching the internet backbone. This is how Netflix delivers petabytes of video every day without melting the internet.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Origin Architecture**: Origin servers behind a CDN see only cache MISS requests (typically 5-20% of total traffic). This dramatically reduces the origin's required capacity. But the origin must handle spikes during cache invalidation (purging popular content) or cold starts.
- **Cache Key Design**: The CDN cache key determines what constitutes a unique cached object. Default is the full URL. But too many variations (query params, device-type headers) can fragment the cache and reduce hit ratio.
- **Cache Invalidation Strategy**: Versioned URLs (recommended) vs. purge-on-deploy vs. short TTLs. Versioned URLs are the gold standard—zero coordination needed between deploy and cache invalidation.
- **SSL/TLS Configuration**: CDN-managed certificates with auto-renewal simplify operations, but some compliance regimes (FedRAMP, certain PCI scenarios) require customer-controlled private keys that CDN providers cannot access.
- **Multi-CDN Strategy**: Large services often use multiple CDN providers for redundancy. If one CDN has an outage (it happens), traffic fails over to another. Requires DNS health checks and consistent cache key configurations across providers.

### Business Impact
- **Page Load Performance**: Walmart found that every 100ms improvement in page load time increased revenue by 1%. Amazon reported a 1% revenue decrease for every 100ms of latency.
- **Bandwidth Cost Reduction**: CDNs reduce origin bandwidth by 80-95% for cacheable content. Cloud egress costs ($0.05-0.12 per GB) add up fast for high-traffic sites. CDN pricing is typically lower than cloud egress.
- **Global Reach**: A server in Virginia cannot serve users in India with acceptable latency. CDN PoPs in Mumbai make the application feel local to Indian users, critical for market expansion.
- **SEO**: Google uses page speed as a ranking factor. CDN-accelerated sites rank higher, driving organic traffic and revenue.

## On-Premises Examples

### Varnish Cache (Reverse Proxy CDN)
Varnish is a high-performance HTTP accelerator often deployed as an on-premises edge cache:

```vcl
# /etc/varnish/default.vcl
vcl 4.0;

backend default {
    .host = "origin.example.com";
    .port = "80";
}

sub vcl_recv {
    # Don't cache authenticated requests
    if (req.http.Authorization) {
        return (pass);
    }
    
    # Strip marketing query params for better cache hit ratio
    set req.url = regsuball(req.url, "([?&])(utm_[^=&]+)=[^&]+", "\1\2=");
}

sub vcl_backend_response {
    # Cache static assets for 30 days
    if (bereq.url ~ "\.(jpg|jpeg|png|gif|css|js|woff2)$") {
        set beresp.ttl = 30d;
        set beresp.http.Cache-Control = "public, max-age=2592000";
    }
    
    # Cache successful API responses for 5 minutes
    if (bereq.url ~ "^/api/" && beresp.status == 200) {
        set beresp.ttl = 5m;
    }
}

sub vcl_deliver {
    # Add cache status header for debugging
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }
}
```

### Apache Traffic Server
High-performance caching proxy used by Yahoo, Comcast, and Apple:

```bash
# records.config
CONFIG proxy.config.http.cache.required_headers INT 0
CONFIG proxy.config.http.cache.range_lookup INT 1
CONFIG proxy.config.cache.ram_cache.size INT 2GB
```

### Squid as Web Cache
```bash
# squid.conf
http_port 3128 accel vhost
cache_peer origin.example.com parent 80 0 no-query originserver

cache_dir ufs /cache 100000 16 256
maximum_object_size 1 GB
refresh_pattern \.(jpg|png|css|js)$ 1440 100% 2880
```

## AWS Examples

### Amazon CloudFront
AWS's global CDN with 600+ PoPs:

```hcl
resource "aws_cloudfront_distribution" "cdn" {
  enabled             = true
  is_ipv6_enabled     = true
  price_class         = "PriceClass_All"  # Use all edge locations
  default_root_object = "index.html"

  # S3 origin for static assets
  origin {
    domain_name = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id   = "S3-static"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }

  # ALB origin for dynamic API traffic with origin shield
  origin {
    domain_name = aws_lb.app.dns_name
    origin_id   = "ALB-app"
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
    
    # Origin shield: regional cache before hitting origin
    origin_shield {
      enabled              = true
      origin_shield_region = "us-east-1"
    }
  }

  # Caching behavior for static assets
  ordered_cache_behavior {
    path_pattern     = "/static/*"
    target_origin_id = "S3-static"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    min_ttl     = 0
    default_ttl = 86400    # 24 hours
    max_ttl     = 31536000 # 1 year
    
    compress = true
    viewer_protocol_policy = "redirect-to-https"
  }

  # Caching behavior for API responses
  ordered_cache_behavior {
    path_pattern     = "/api/products/*"
    target_origin_id = "ALB-app"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    
    forwarded_values {
      query_string = true
      cookies {
        forward = "none"
      }
      headers = ["Origin"]  # vary cache by Origin for CORS
    }
    
    min_ttl     = 0
    default_ttl = 60      # 1 minute cache for product data
    max_ttl     = 300     # 5 minutes max
  }

  # Lambda@Edge for request manipulation at edge
  # (e.g., normalize cache keys, A/B testing, authentication)
  default_cache_behavior {
    target_origin_id = "ALB-app"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    
    forwarded_values {
      query_string = true
      cookies {
        forward = "all"
      }
    }
    
    lambda_function_association {
      event_type   = "origin-request"
      lambda_arn   = aws_lambda_function.edge_function.qualified_arn
      include_body = false
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cdn.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
}

# Lambda@Edge for URL rewriting / cache normalization
resource "aws_lambda_function" "edge_function" {
  filename      = "edge.zip"
  function_name = "cdn-edge-processor"
  role          = aws_iam_role.edge_lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  publish       = true
}
```

```javascript
// Lambda@Edge: normalize cache keys by stripping marketing params
exports.handler = async (event) => {
    const request = event.Records[0].cf.request;
    const url = new URL(request.uri, `https://${request.headers.host[0].value}`);
    
    // Remove utm_ params from cache key
    const params = url.searchParams;
    const marketingParams = ['utm_source', 'utm_medium', 'utm_campaign', 'utm_term', 'utm_content'];
    marketingParams.forEach(p => params.delete(p));
    
    request.uri = url.pathname + (params.toString() ? '?' + params.toString() : '');
    return request;
};
```

### CloudFront Functions (Lighter than Lambda@Edge)
For simple request/response manipulation at sub-ms latency:

```javascript
// Cache key normalization: sort query params for better hit ratio
function handler(event) {
    var request = event.request;
    var queryString = request.querystring;
    
    if (queryString && queryString.length > 0) {
        var params = queryString.split('&').sort().join('&');
        request.querystring = params;
    }
    
    return request;
}
```

### AWS Global Accelerator
Not a CDN per se, but accelerates TCP/UDP traffic by routing through AWS's global network backbone:

```hcl
resource "aws_globalaccelerator_accelerator" "app" {
  name            = "app-accelerator"
  ip_address_type = "IPV4"
  enabled         = true
}

resource "aws_globalaccelerator_listener" "app" {
  accelerator_arn = aws_globalaccelerator_accelerator.app.id
  client_affinity = "SOURCE_IP"
  protocol        = "TCP"
  port_range {
    from_port = 443
    to_port   = 443
  }
}
```

## GCP Examples

### Cloud CDN
Integrated with Cloud Load Balancing, uses Google's global edge network:

```bash
# Create backend bucket with CDN enabled
gcloud compute backend-buckets create static-backend \
  --gcs-bucket-name=my-static-bucket \
  --enable-cdn

# Create URL map with CDN
gcloud compute url-maps create web-map \
  --default-backend-bucket=static-backend

# Cache configuration
gcloud compute backend-buckets update static-backend \
  --custom-response-header='Cache-Hit: {cdn_cache_status}' \
  --default-ttl=3600 \
  --max-ttl=86400
```

```bash
# CDN cache invalidation
gcloud compute url-maps invalidate-cdn-cache web-map \
  --path "/images/updated-banner.jpg"
```

### Media CDN
Purpose-built for large-scale streaming and download delivery (YouTube-scale):

```bash
gcloud edge-cache services create video-cdn \
  --routing-path="*" \
  --origin="gs://my-video-bucket"
```

### Cloud Load Balancing with CDN
```yaml
# Cloud CDN with signed URLs for restricted content
gcloud compute backend-services update my-backend \
  --enable-cdn \
  --cache-key-include-host \
  --cache-key-include-protocol \
  --cache-key-include-query-string=true \
  --cache-key-whitelist="version,token"
```

### Cloud CDN Signed URLs
```python
import base64
import hashlib
import hmac
import time

def generate_signed_url(url, key_name, base64_key, expiration_seconds=3600):
    """Generate a Cloud CDN signed URL for restricted content"""
    expiration = int(time.time()) + expiration_seconds
    
    # Build the signature string
    url_parts = url.split('?')
    signature_input = f"{url_parts[0]}?Expires={expiration}&KeyName={key_name}"
    
    # Sign with HMAC-SHA1
    key = base64.urlsafe_b64decode(base64_key)
    signature = hmac.new(key, signature_input.encode(), hashlib.sha1).digest()
    encoded_signature = base64.urlsafe_b64encode(signature).decode().rstrip('=')
    
    return f"{signature_input}&Signature={encoded_signature}"

# Signed URL: expires after set time, prevents hotlinking
url = generate_signed_url(
    "https://cdn.example.com/premium/video.mp4",
    "my-signing-key",
    "BASE64_ENCODED_KEY"
)
```

## Azure Examples

### Azure Front Door (Global Anycast + CDN)
Layer 7 global load balancer with CDN capabilities:

```bash
az afd profile create \
  --profile-name myCDNProfile \
  --resource-group myResourceGroup \
  --sku Premium_AzureFrontDoor

az afd endpoint create \
  --endpoint-name myEndpoint \
  --profile-name myCDNProfile \
  --resource-group myResourceGroup
```

```json
{
  "name": "caching-rule",
  "properties": {
    "routeConfiguration": {
      "@odata.type": "#Microsoft.Azure.FrontDoor.Models.FrontdoorForwardingConfiguration",
      "forwardingProtocol": "HttpsOnly",
      "cacheConfiguration": {
        "queryParameterStripDirective": "StripAllExcept",
        "queryParameters": "version,country",
        "dynamicCompression": "Enabled",
        "cacheDuration": "1.00:00:00"
      }
    }
  }
}
```

### Azure CDN (from Verizon/Akamai)
```bash
az cdn profile create \
  --name myCDN \
  --resource-group myResourceGroup \
  --sku Standard_Verizon

az cdn endpoint create \
  --name myEndpoint \
  --profile-name myCDN \
  --resource-group myResourceGroup \
  --origin myapp.azurewebsites.net \
  --origin-host-header myapp.azurewebsites.net

# Custom domain with CDN-managed HTTPS
az cdn custom-domain create \
  --endpoint-name myEndpoint \
  --profile-name myCDN \
  --resource-group myResourceGroup \
  --custom-domain-name cdn.example.com \
  --hostname cdn.example.com

# Enable HTTPS
az cdn custom-domain enable-https \
  --endpoint-name myEndpoint \
  --profile-name myCDN \
  --resource-group myResourceGroup \
  --custom-domain-name cdn.example.com
```

### Azure CDN Rules Engine
```json
{
  "rules": [
    {
      "name": "cache-static-assets",
      "order": 1,
      "conditions": [
        {"name": "UrlFileExtension", "parameters": {"operator": "Equal", "matchValues": ["jpg","png","css","js","woff2"]}}
      ],
      "actions": [
        {"name": "ModifyResponseHeader", "parameters": {"headerAction": "Overwrite", "headerName": "Cache-Control", "value": "public, max-age=2592000"}},
        {"name": "ModifyResponseHeader", "parameters": {"headerAction": "Append", "headerName": "X-Cache-Status", "value": "{cdn_cache_status}"}}
      ]
    },
    {
      "name": "redirect-www",
      "order": 2,
      "conditions": [
        {"name": "RequestHeader", "parameters": {"operator": "Wildcard", "selector": "host", "matchValues": ["example.com"]}}
      ],
      "actions": [
        {"name": "UrlRedirect", "parameters": {"redirectProtocol": "Https", "redirectType": "Moved", "destinationHostname": "www.example.com"}}
      ]
    }
  ]
}
```

### Azure CDN Token Authentication
```python
import hashlib
import base64
from datetime import datetime, timedelta

def generate_azure_cdn_token(path, key, expiration_hours=1):
    """Generate Azure CDN token for secure content delivery"""
    expires = int((datetime.utcnow() + timedelta(hours=expiration_hours)).timestamp())
    
    # Token format: ec_expire=EXPIRES&ec_url_allow=PATH
    token_string = f"ec_expire={expires}&ec_url_allow={path}"
    
    # Hash with SHA256
    signature = hashlib.sha256((token_string + key).encode()).digest()
    encoded_sig = base64.urlsafe_b64encode(signature).decode().rstrip('=')
    
    return f"{token_string}&ec_signature={encoded_sig}"

# Full CDN URL with token
token = generate_azure_cdn_token("/videos/premium/*", "secret-key")
secure_url = f"https://cdn.example.com/videos/premium/intro.mp4?{token}"
```

## Summary Decision Matrix

| CDN Capability | AWS CloudFront | GCP Cloud CDN | Azure Front Door / CDN |
|---------------|----------------|---------------|----------------------|
| Edge locations | 600+ PoPs | 130+ PoPs (Google backbone) | 130+ PoPs |
| Origin Shield | Yes (regional cache) | No explicit (but multi-tier) | Yes (caching rules) |
| Edge compute | Lambda@Edge + CloudFront Functions | N/A (use Cloud Run near regions) | Azure Functions @ Edge |
| Real-time logs | Yes (Kinesis Data Stream) | Yes (Cloud Logging) | Yes (Event Hub) |
| Signed URLs/Cookies | Yes | Yes | Yes (token auth) |
| WAF integration | AWS WAF | Cloud Armor | Azure WAF / Front Door WAF |
| DDoS protection | AWS Shield (+Advanced) | Cloud Armor | Azure DDoS Protection |
| Price model | Per request + GB transferred | Per request + GB transferred | Per GB transferred (+ rules) |

CDNs are no longer optional—they are the default deployment pattern for any publicly accessible web application. The combination of latency reduction, origin offload, and built-in security makes them the first line of defense and the first optimization for global user experience. A solution architect should design applications from day one with CDN integration in mind: versioned URLs, appropriate cache headers, and edge-optimized cache key strategies.
