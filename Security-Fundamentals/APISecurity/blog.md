# API Security

## Introduction
API Security has become the most critical security discipline in modern application architecture. As applications evolved from monolithic server-rendered HTML to API-first architectures (REST, GraphQL, gRPC), the attack surface shifted dramatically. What was once protected by server-side session state behind a single domain is now a collection of stateless APIs accessible from mobile apps, SPAs, third-party integrations, and IoT devices.

API attacks increased 681% between 2021 and 2023 (Salt Security). Unlike traditional web attacks that target HTML forms and cookies, API attacks exploit authentication tokens, excessive data exposure, broken object-level authorization, and rate limit bypasses. API security requires a fundamentally different approach from traditional web security, focused on authentication, authorization, input validation, and rate limiting at the endpoint level.

## Definition
**API Security** is the practice of protecting APIs from attacks, unauthorized access, and data breaches throughout their lifecycle—design, development, deployment, and runtime. It encompasses:

- **Authentication**: Verifying the identity of API consumers (API keys, OAuth 2.0 tokens, JWT, mTLS)
- **Authorization**: Ensuring authenticated consumers can only access permitted resources (scopes, claims, resource-level permissions)
- **Input Validation**: Preventing injection attacks, parameter tampering, and malformed payloads
- **Rate Limiting**: Controlling request frequency to prevent abuse and ensure fair resource allocation
- **Data Protection**: Encrypting data in transit (TLS), minimizing data exposure in responses
- **Monitoring & Threat Detection**: Logging API calls, detecting anomalies, and alerting on attacks

## Concept Explanation

### API Authentication Patterns

#### API Keys (Simplest, Least Secure)
```python
# Server-side validation
API_KEYS = {
    'sk_live_abc123': {'client': 'Web App', 'rate_limit': 100},
    'sk_live_xyz789': {'client': 'Mobile App', 'rate_limit': 50}
}

@app.before_request
def validate_api_key():
    api_key = request.headers.get('X-API-Key')
    if not api_key or api_key not in API_KEYS:
        return jsonify({'error': 'Invalid API key'}), 401
    g.client_info = API_KEYS[api_key]
```

**Pros**: Simple, easy to implement. **Cons**: Static secret, no expiration, no granular permissions, easily leaked. Only suitable for low-risk, internal, or rate-limiting-only scenarios.

#### JWT Bearer Tokens (Most Common for User-Facing APIs)
```python
import jwt
from functools import wraps

def require_auth(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get('Authorization', '').replace('Bearer ', '')
        if not token:
            return jsonify({'error': 'Missing token'}), 401
        
        try:
            decoded = jwt.decode(
                token,
                PUBLIC_KEY,  # or SECRET_KEY for HS256
                algorithms=['RS256'],
                audience='https://api.example.com',
                options={'require': ['exp', 'sub', 'aud']}
            )
            g.current_user = decoded
        except jwt.ExpiredSignatureError:
            return jsonify({'error': 'Token expired'}), 401
        except jwt.InvalidTokenError as e:
            return jsonify({'error': f'Invalid token: {str(e)}'}), 401
        
        return f(*args, **kwargs)
    return decorated

@app.route('/api/orders')
@require_auth
def get_orders():
    user_id = g.current_user['sub']
    orders = Order.query.filter_by(customer_id=user_id).all()
    return jsonify([o.to_dict() for o in orders])
```

#### OAuth 2.0 with Scopes (Fine-Grained Authorization)
```python
from functools import wraps

def require_scope(*required_scopes):
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            token_scopes = g.current_user.get('scope', '').split()
            if not set(required_scopes).issubset(set(token_scopes)):
                return jsonify({
                    'error': 'insufficient_scope',
                    'required': list(required_scopes),
                    'provided': token_scopes
                }), 403
            return f(*args, **kwargs)
        return decorated
    return decorator

@app.route('/api/orders')
@require_auth
@require_scope('read:orders')
def get_orders():
    ...

@app.route('/api/orders', methods=['POST'])
@require_auth
@require_scope('write:orders')
def create_order():
    ...

@app.route('/api/admin/users')
@require_auth
@require_scope('admin:users')
def list_users():
    ...
```

### OWASP API Security Top 10 (2023)

1. **Broken Object Level Authorization (BOLA)**: API exposes object IDs; attacker iterates to access others' data
2. **Broken Authentication**: Weak or misconfigured authentication allowing credential stuffing, token forgery
3. **Broken Object Property Level Authorization**: Mass assignment—attacker sets fields they shouldn't (e.g., `"role":"admin"`)
4. **Unrestricted Resource Consumption**: No rate limiting leads to DoS and cost amplification
5. **Broken Function Level Authorization**: User can call admin endpoints because of missing authorization checks
6. **Unrestricted Access to Sensitive Business Flows**: Automated abuse of business logic (e.g., scalping limited items)
7. **Server-Side Request Forgery (SSRF)**: API fetches attacker-supplied URLs
8. **Security Misconfiguration**: Verbose errors, unnecessary HTTP methods, CORS misconfiguration
9. **Improper Inventory Management**: Exposed old/ debug/ beta API versions with no security
10. **Unsafe Consumption of APIs**: Trusting data from third-party APIs without validation

### Defending Against API Attacks

#### BOLA Protection (Object-Level Authorization)
```python
# VULNERABLE: No ownership check
@app.route('/api/orders/<order_id>')
def get_order(order_id):
    return Order.query.get(order_id).to_dict()

# SECURE: Always verify ownership
@app.route('/api/orders/<order_id>')
@require_auth
def get_order(order_id):
    order = Order.query.get_or_404(order_id)
    if order.customer_id != g.current_user['sub']:
        # Return 404, not 403, to avoid leaking existence of other users' orders
        abort(404)
    return order.to_dict()
```

#### Mass Assignment Protection
```python
# VULNERABLE: Accepts all fields from request
@app.route('/api/users/<user_id>', methods=['PATCH'])
def update_user(user_id):
    user = User.query.get(user_id)
    for key, value in request.json.items():
        setattr(user, key, value)  # Attacker sets {"role": "admin"}
    db.session.commit()

# SECURE: Explicit allowlist of updatable fields
ALLOWED_UPDATE_FIELDS = {'name', 'email', 'phone', 'preferences'}

@app.route('/api/users/<user_id>', methods=['PATCH'])
def update_user(user_id):
    user = User.query.get(user_id)
    for key, value in request.json.items():
        if key not in ALLOWED_UPDATE_FIELDS:
            continue  # Silently ignore disallowed fields
        setattr(user, key, value)
    db.session.commit()
```

#### GraphQL-Specific Protections
```python
# Depth limiting
from graphql import validate, parse
from graphql.validation.rules import depth_limit

MAX_QUERY_DEPTH = 5

@app.route('/graphql', methods=['POST'])
def graphql():
    query = request.json['query']
    document = parse(query)
    
    validation_errors = validate(schema, document, [depth_limit(MAX_QUERY_DEPTH)])
    if validation_errors:
        return jsonify({'errors': [str(e) for e in validation_errors]}), 400

# Rate limiting by query complexity
def calculate_complexity(document):
    cost = 0
    for definition in document.definitions:
        for field in definition.selection_set.selections:
            cost += 1
            # Multiply by pagination count if present
            for arg in field.arguments:
                if arg.name.value == 'first' and int(arg.value.value) > 100:
                    return float('inf')  # Too expensive, reject
    return cost
```

### API Rate Limiting

#### Token Bucket Per User/API Key
```python
import redis
import time

class APIRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def check(self, client_id, endpoint, max_req, window_sec):
        key = f"ratelimit:{client_id}:{endpoint}"
        current = int(self.redis.get(key) or 0)
        
        if current >= max_req:
            ttl = self.redis.ttl(key)
            return False, ttl  # rate limited, retry after ttl seconds
        
        pipe = self.redis.pipeline()
        pipe.incr(key)
        if current == 0:
            pipe.expire(key, window_sec)
        pipe.execute()
        return True, 0

# Usage with tiered limits
rate_limiter = APIRateLimiter(redis_client)

LIMITS = {
    'free': {'/api/orders': (10, 60), '/api/search': (30, 60)},
    'pro': {'/api/orders': (100, 60), '/api/search': (300, 60)},
    'enterprise': {'/api/orders': (1000, 60), '/api/search': (5000, 60)}
}

@app.before_request
def check_rate_limit():
    tier = g.client_info.get('tier', 'free')
    limits = LIMITS[tier]
    endpoint_limits = limits.get(request.path)
    
    if endpoint_limits:
        max_req, window = endpoint_limits
        allowed, retry_after = rate_limiter.check(
            g.client_info['api_key'],
            request.path,
            max_req, window
        )
        if not allowed:
            response = jsonify({'error': 'Rate limit exceeded'})
            response.status_code = 429
            response.headers['Retry-After'] = str(retry_after)
            response.headers['X-RateLimit-Limit'] = str(max_req)
            response.headers['X-RateLimit-Remaining'] = '0'
            return response
```

### API Gateway Security Patterns

```yaml
# API Gateway centralized security
Gateway:
  Authentication: Token validation, API key verification
  Authorization:  Scope checking, IP allowlisting
  Rate Limiting:  Per-client, per-endpoint throttling
  Transformation: Strip sensitive fields, add security headers
  Monitoring:     Request logging, anomaly detection

Backend Services: (behind gateway, not directly accessible)
  - Receives validated, authenticated requests
  - Trusts headers set by gateway (X-User-ID, X-Scopes)
  - Focused on business logic, not security plumbing
```

## Layman's Explanation

### API Security: The Hotel Key Card System
A hotel (API) has hundreds of rooms (resources) and thousands of guests (API consumers).

**API Key**: A guest checks in and gets a room number. Anyone who knows the room number can try to enter. Simple, but if someone overhears the number (key leaks), they can enter.

**JWT Token**: The hotel uses electronic key cards (JWTs). Each card has the guest's name (subject), room number (user ID), and expiry date encoded in it. The card is cryptographically signed so it can't be forged. When the card expires, the guest must get a new one (token refresh).

**OAuth Scopes**: The key card also has access levels. A "guest" card opens their room and the gym. A "housekeeping" card opens all rooms but only between 9am-5pm. A "manager" card opens everything, including the cash room.

**BOLA Attack**: Guest in Room 301 tries their key in Room 302. A secure hotel (proper authorization) checks not just "is this a valid key?" but also "does this key belong to Room 302?" The lock says "no" (404 Not Found, to not reveal that Room 302 exists).

**Mass Assignment**: A guest fills out a registration form and adds `"VIP status": "Platinum"` as an extra field. A naive system saves this directly to the database. A secure system only saves fields that guests are allowed to set (name, email, preferences—not VIP status).

## Why Solution Architects Must Acquire This  

### Critical Design Decisions
- **API Gateway vs Service-Level Security**: Centralized security at the API gateway simplifies management, ensures consistency, and prevents gaps. But some authorization decisions require deep business logic only the service knows (can user X access order Y if they're in the same department?). Hybrid: gateway handles authentication + coarse authorization (valid token, basic scopes); services handle fine-grained authorization.
- **Token Validation Architecture**: Validate JWT signatures locally (using cached JWKS) vs remotely (introspection endpoint). Local validation is faster and survives auth server outages. Remote validation ensures tokens haven't been revoked. Hybrid: local validation with short token lifetimes (5 min) + remote refresh.
- **API Version Deprecation**: Old API versions accumulate and are often less secure. v1 might allow plain HTTP, have verbose errors, or miss authorization checks that v2 added. API version lifecycle management (deprecation announcements, sunset dates, forced upgrade) is a security practice.
- **East-West API Security**: Service-to-service communication inside the cluster needs the same security rigor as north-south (client-to-API). mTLS or service mesh (Istio/Consul) ensures encrypted, authenticated inter-service communication. Zero-trust: every service-to-service call is authenticated.

### Business Impact
- **Data Breach Prevention**: API attacks account for an increasing percentage of data breaches. The Optus breach (2022, 11 million customers) was an API that didn't require authentication. Proper API auth would have prevented it.
- **Revenue Protection**: Rate-limited APIs prevent abuse that drives cloud costs. An un-throttled image generation API can be called 100,000 times by a single user, costing thousands in compute.
- **Partner Ecosystem**: Companies expose APIs to partners and third-party developers. Strong API security (OAuth, rate limiting, developer portal with documentation) is a business enabler—partners integrate with confidence.
- **Compliance**: PCI DSS 4.0 explicitly requires API security controls (Req 6.2.4). GDPR data minimization requires APIs to return only necessary data, not entire database rows.

## On-Premises Examples

### Kong API Gateway Security
```yaml
# kong.yml
services:
  - name: order-service
    url: http://order-service:8080
    routes:
      - name: orders
        paths: ["/api/orders"]
        methods: ["GET", "POST"]
        strip_path: false
    
    plugins:
      - name: jwt
        config:
          claims_to_verify:
            - exp
          maximum_expiration: 3600
      
      - name: rate-limiting
        config:
          minute: 100
          hour: 5000
          policy: redis
          redis_host: redis-cluster
          redis_port: 6379
          fault_tolerant: true
      
      - name: request-size-limiting
        config:
          allowed_payload_size: 10  # MB
      
      - name: cors
        config:
          origins: ["https://app.example.com"]
          methods: ["GET", "POST"]
          headers: ["Authorization", "Content-Type"]
          credentials: true
          max_age: 3600
      
      - name: ip-restriction
        config:
          deny: ["0.0.0.0/0"]  # Deny all by default
          allow: ["10.0.0.0/8", "172.16.0.0/12"]  # Allow internal
```

### Flask API Security Middleware
```python
from flask import Flask, request, jsonify, g
from flask_talisman import Talisman
from flask_cors import CORS
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

app = Flask(__name__)

# Security headers
Talisman(app, 
    force_https=True,
    strict_transport_security=True,
    session_cookie_secure=True,
    content_security_policy={
        'default-src': "'none'",
        'frame-ancestors': "'none'"
    }
)

# CORS: only allow specific origins
CORS(app, origins=['https://app.example.com'])

# Rate limiting
limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"],
    storage_uri="redis://localhost:6379"
)

@app.after_request
def add_security_headers(response):
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '0'  # Deprecated, CSP handles this
    response.headers['Cache-Control'] = 'no-store'
    response.headers['Pragma'] = 'no-cache'
    response.headers.pop('Server', None)  # Don't leak server version
    return response
```

### GraphQL Security (Apollo Server)
```javascript
const { ApolloServer } = require('apollo-server');
const { createRateLimitDirective } = require('graphql-rate-limit');
const depthLimit = require('graphql-depth-limit');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  
  // Query depth limit (prevents deeply nested attacks)
  validationRules: [depthLimit(5)],
  
  // Schema directives for security
  schemaDirectives: {
    rateLimit: createRateLimitDirective({
      identifyContext: (ctx) => ctx.user?.id || ctx.request.ip
    }),
    auth: AuthDirective,
  },
  
  // Disable introspection in production
  introspection: process.env.NODE_ENV !== 'production',
  
  // Disable playground in production
  playground: process.env.NODE_ENV !== 'production',
  
  context: ({ req }) => {
    const token = req.headers.authorization?.replace('Bearer ', '');
    const user = verifyToken(token);
    return { user, req };
  }
});
```

## AWS Examples

### Amazon API Gateway Security
```yaml
# API Gateway with Lambda authorizer
ApiGatewayRestApi:
  Type: AWS::Serverless::Api
  Properties:
    StageName: prod
    Auth:
      DefaultAuthorizer: TokenAuthorizer
      Authorizers:
        TokenAuthorizer:
          FunctionArn: !GetAtt AuthFunction.Arn
          Identity:
            Header: Authorization
            ValidationExpression: '^Bearer [-0-9a-zA-Z\._]*$'
      InvokeRole: NONE  # Don't use caller credentials
    GatewayResponses:
      UNAUTHORIZED:
        StatusCode: 401
        ResponseParameters:
          Headers:
            Access-Control-Allow-Origin: "'https://app.example.com'"
      ACCESS_DENIED:
        StatusCode: 403
      RESOURCE_NOT_FOUND:
        StatusCode: 404
```

```python
# Lambda custom authorizer
import jwt
import requests
import os

def lambda_handler(event, context):
    token = event['authorizationToken'].replace('Bearer ', '')
    
    try:
        # Decode and verify JWT
        decoded = jwt.decode(
            token,
            options={'verify_signature': False},  # Will verify below
            algorithms=['RS256']
        )
        
        # Fetch JWKS and verify signature
        jwks_client = jwt.PyJWKClient(os.environ['JWKS_URL'])
        signing_key = jwks_client.get_signing_key_from_jwt(token)
        
        decoded = jwt.decode(
            token,
            signing_key.key,
            algorithms=['RS256'],
            audience=os.environ['AUDIENCE'],
            options={'require': ['exp', 'sub', 'aud']}
        )
        
        # Generate IAM policy
        return generate_policy(decoded['sub'], 'Allow', event['methodArn'], decoded)
        
    except jwt.InvalidTokenError:
        return generate_policy('user', 'Deny', event['methodArn'])

def generate_policy(principal_id, effect, resource, context=None):
    policy = {
        'principalId': principal_id,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [{
                'Action': 'execute-api:Invoke',
                'Effect': effect,
                'Resource': resource
            }]
        }
    }
    if context:
        policy['context'] = {
            'userId': context.get('sub', ''),
            'scopes': context.get('scope', '')
        }
    return policy
```

### API Gateway Usage Plans + API Keys
```hcl
resource "aws_api_gateway_usage_plan" "pro" {
  name = "pro-plan"

  api_stages {
    api_id = aws_api_gateway_rest_api.main.id
    stage  = "prod"
  }

  throttle_settings {
    burst_limit = 200
    rate_limit  = 100
  }

  quota_settings {
    limit  = 1000000
    period = "MONTH"
  }
}

resource "aws_api_gateway_api_key" "client_a" {
  name = "client-a"
}

resource "aws_api_gateway_usage_plan_key" "client_a" {
  key_id        = aws_api_gateway_api_key.client_a.id
  key_type      = "API_KEY"
  usage_plan_id = aws_api_gateway_usage_plan.pro.id
}
```

### AWS WAF for API Protection
```hcl
resource "aws_wafv2_web_acl" "api" {
  name  = "api-waf"
  scope = "REGIONAL"

  default_action {
    block {}
  }

  # Only allow POST and GET (principle of least privilege)
  rule {
    name     = "method-restriction"
    priority = 0

    action {
      allow {}
    }

    statement {
      or_statement {
        statement {
          byte_match_statement {
            search_string = "GET"
            field_to_match { method {} }
            text_transformation { priority = 0; type = "NONE" }
            positional_constraint = "EXACTLY"
          }
        }
        statement {
          byte_match_statement {
            search_string = "POST"
            field_to_match { method {} }
            text_transformation { priority = 0; type = "NONE" }
            positional_constraint = "EXACTLY"
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "MethodRestriction"
      sampled_requests_enabled   = true
    }
  }
}
```

## GCP Examples

### Cloud Endpoints (API Management)
```yaml
# OpenAPI spec with security definitions
swagger: '2.0'
securityDefinitions:
  api_key:
    type: "apiKey"
    name: "key"
    in: "query"
  auth0_jwt:
    authorizationUrl: ""
    flow: "implicit"
    type: "oauth2"
    x-google-issuer: "https://auth.example.com"
    x-google-jwks_uri: "https://auth.example.com/.well-known/jwks.json"
    x-google-audiences: "https://api.example.com"

paths:
  /orders:
    get:
      security:
        - auth0_jwt: ["read:orders"]
      x-google-quota:
        metricCosts:
          "read-requests": 1
      ...
    post:
      security:
        - auth0_jwt: ["write:orders"]
      ...
```

### Apigee API Security
```xml
<policies>
  <!-- Verify API Key -->
  <VerifyAPIKey async="false" continueOnError="false" enabled="true" name="Verify-API-Key">
    <APIKey ref="request.queryparam.apikey"/>
  </VerifyAPIKey>
  
  <!-- OAuth v2 token validation -->
  <OAuthV2 async="false" continueOnError="false" enabled="true" name="Validate-Access-Token">
    <Operation>VerifyAccessToken</Operation>
    <SupportedGrantTypes>
      <GrantType>client_credentials</GrantType>
    </SupportedGrantTypes>
  </OAuthV2>
  
  <!-- JSON threat protection -->
  <JSONThreatProtection async="false" continueOnError="false" enabled="true" name="JSON-Threat-Protection">
    <ArrayElementCount>100</ArrayElementCount>
    <ContainerDepth>5</ContainerDepth>
    <ObjectEntryCount>100</ObjectEntryCount>
    <ObjectEntryNameLength>50</ObjectEntryNameLength>
    <StringValueLength>10000</StringValueLength>
  </JSONThreatProtection>
  
  <!-- Spike arrest (rate limiting) -->
  <SpikeArrest async="false" continueOnError="false" enabled="true" name="Spike-Arrest">
    <Rate>30ps</Rate>  <!-- 30 requests per second -->
    <UseEffectiveCount>true</UseEffectiveCount>
  </SpikeArrest>
</policies>
```

## Azure Examples

### Azure API Management Security
```xml
<policies>
    <inbound>
        <base />
        
        <!-- JWT validation -->
        <validate-jwt 
            header-name="Authorization" 
            failed-validation-httpcode="401" 
            failed-validation-error-message="Unauthorized" 
            require-expiration-time="true" 
            require-scheme="Bearer"
            require-signed-tokens="true">
            <openid-config url="https://login.microsoftonline.com/tenant/.well-known/openid-configuration" />
            <audiences>
                <audience>api://my-api-client-id</audience>
            </audiences>
            <issuers>
                <issuer>https://login.microsoftonline.com/tenant/v2.0</issuer>
            </issuers>
        </validate-jwt>
        
        <!-- Rate limiting by subscription key -->
        <rate-limit-by-key 
            calls="100" 
            renewal-period="60" 
            counter-key="@(context.Subscription.Id)"
            increment-condition="@(context.Response.StatusCode >= 200 && context.Response.StatusCode < 300)" />
        
        <!-- IP filtering -->
        <ip-filter action="forbid">
            <address-range from="0.0.0.0" to="255.255.255.255" />
        </ip-filter>
        <ip-filter action="allow">
            <address>10.0.0.0/8</address>
            <address>172.16.0.0/12</address>
        </ip-filter>
        
        <!-- CORS -->
        <cors allow-credentials="true">
            <allowed-origins>
                <origin>https://app.example.com</origin>
            </allowed-origins>
            <allowed-methods>
                <method>GET</method>
                <method>POST</method>
            </allowed-methods>
        </cors>
        
        <!-- Request size limit -->
        <set-body>
            <@ {
                var body = context.Request.Body.As<string>(preserveContent: true);
                if (body.Length > 1024 * 1024 * 10) {  // 10MB
                    return Response(413, "Payload too large");
                }
                return body;
            }
        </set-body>
    </inbound>
    
    <on-error>
        <return-response>
            <set-status code="500" />
            <set-body>Internal server error</set-body>
        </return-response>
    </on-error>
</policies>
```

### Azure AD OAuth 2.0 Protected API
```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

# Using Azure AD for API authentication
import msal
import requests

# Confidential client
app = msal.ConfidentialClientApplication(
    client_id=CLIENT_ID,
    client_credential=CLIENT_SECRET,
    authority="https://login.microsoftonline.com/TENANT_ID"
)

# Acquire token for downstream API
result = app.acquire_token_for_client(scopes=["api://downstream-api/.default"])
access_token = result['access_token']

# Call protected API
response = requests.get(
    "https://api.example.com/orders",
    headers={"Authorization": f"Bearer {access_token}"}
)
```

## Summary Decision Matrix

| API Security Control | Simple (API Keys) | Medium (JWT) | Enterprise (OAuth 2.0 + Gateway) |
|---------------------|-------------------|--------------|----------------------------------|
| Authentication | Static key | Signed JWT with expiry | OAuth 2.0 with PKCE, token rotation |
| Authorization | None (pass/fail) | Claims-based | Scopes + resource-level (BOLA protection) |
| Rate Limiting | Per-IP | Per-user/API key | Tiered plans, per-endpoint, burst handling |
| Input Validation | Minimal | JSON schema | API schema validation + WAF rules |
| Data Protection | TLS | TLS + JWT encryption | TLS + encrypted payloads + data minimization |
| Monitoring | Access logs | Logs + basic metrics | Full API analytics + anomaly detection |

API security must be designed, not bolted on. It starts with authentication (who is calling?), requires authorization (what are they allowed to do?), enforces rate limiting (how much can they call?), validates input (is the request safe?), and monitors continuously (are we under attack?). An API gateway centralizes these concerns, but fine-grained authorization and threat detection must extend to every service. The architect's role is to ensure that every API endpoint—public, partner, or internal—has defense in depth at every layer.
