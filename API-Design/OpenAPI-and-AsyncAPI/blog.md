# OpenAPI & AsyncAPI

## Introduction
OpenAPI (formerly Swagger) and AsyncAPI are specification standards for describing APIs. OpenAPI describes synchronous REST APIs—endpoints, request/response schemas, authentication. AsyncAPI describes event-driven APIs—message brokers, channels, publish/subscribe semantics. Together, they provide machine-readable API contracts that enable code generation, documentation, validation, testing, and client SDK generation.

OpenAPI has become the universal standard for REST API description, supported across the entire API lifecycle toolchain. AsyncAPI is its event-driven counterpart, addressing the gap in event-driven architecture documentation. For solution architects, these specifications are not just documentation—they are the executable contract between teams, the input to security scanning tools, and the source of truth for API gateways.

## Definition

**OpenAPI Specification (OAS)** is a standard, language-agnostic interface description for REST APIs, allowing both humans and computers to discover and understand the capabilities of a service without access to source code. Current version: 3.1.

**AsyncAPI** is an open-source specification for describing asynchronous APIs—message-driven architectures, event-driven architectures, and streaming APIs. It is heavily inspired by OpenAPI but adapted for event-driven communication.

## OpenAPI

### Basic Structure

```yaml
openapi: 3.1.0
info:
  title: Order Management API
  version: 1.2.0
  description: API for creating and managing customer orders
  contact:
    name: Order Engineering Team
    email: order-eng@company.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

tags:
  - name: Orders
    description: Order management operations

paths:
  /orders:
    post:
      tags: [Orders]
      summary: Create a new order
      operationId: createOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          $ref: '#/components/responses/ValidationError'
        '401':
          $ref: '#/components/responses/Unauthorized'
      security:
        - OAuth2: [write:orders]

    get:
      tags: [Orders]
      summary: List orders
      operationId: listOrders
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, confirmed, shipped, delivered]
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: Orders retrieved
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderList'

components:
  schemas:
    CreateOrderRequest:
      type: object
      required: [customerId, items]
      properties:
        customerId:
          type: string
          format: uuid
        items:
          type: array
          minItems: 1
          maxItems: 100
          items:
            $ref: '#/components/schemas/OrderItemInput'
    
    Order:
      type: object
      properties:
        id:
          type: string
          format: uuid
        customerId:
          type: string
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        total:
          type: number
          format: double
        status:
          $ref: '#/components/schemas/OrderStatus'
        createdAt:
          type: string
          format: date-time

    OrderStatus:
      type: string
      enum: [pending, confirmed, shipped, delivered, cancelled]

  securitySchemes:
    OAuth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/authorize
          tokenUrl: https://auth.example.com/token
          scopes:
            read:orders: Read orders
            write:orders: Create and update orders

  responses:
    ValidationError:
      description: Validation failed
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    Unauthorized:
      description: Authentication required
```

### Code Generation from OpenAPI

```bash
# Generate server stubs
openapi-generator generate \
  -i openapi.yaml \
  -g python-flask \
  -o ./server-stub

# Generate client SDK
openapi-generator generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./client-sdk

# Generate documentation
redoc-cli bundle openapi.yaml -o docs.html

# Validate spec
spectral lint openapi.yaml
```

## AsyncAPI

### Event-Driven API Description

```yaml
asyncapi: 3.0.0
info:
  title: Order Events API
  version: 1.0.0
  description: Event-driven API for order processing

servers:
  production:
    host: kafka.internal.com:9092
    protocol: kafka
    description: Production Kafka cluster

channels:
  order/placed:
    address: order.placed
    messages:
      OrderPlaced:
        $ref: '#/components/messages/OrderPlaced'
    description: Emitted when a new order is placed

  order/shipped:
    address: order.shipped
    messages:
      OrderShipped:
        $ref: '#/components/messages/OrderShipped'

operations:
  onOrderPlaced:
    action: receive
    channel:
      $ref: '#/channels/order/placed'
    messages:
      - $ref: '#/components/messages/OrderPlaced'

  sendOrderShipped:
    action: send
    channel:
      $ref: '#/channels/order/shipped'
    messages:
      - $ref: '#/components/messages/OrderShipped'

components:
  messages:
    OrderPlaced:
      payload:
        type: object
        required: [orderId, customerId, items, total]
        properties:
          orderId:
            type: string
          customerId:
            type: string
          items:
            type: array
            items:
              type: object
              properties:
                sku: { type: string }
                quantity: { type: integer }
                price: { type: number }
          total:
            type: number
          timestamp:
            type: string
            format: date-time
      headers:
        type: object
        properties:
          correlationId:
            type: string
          messageId:
            type: string

    OrderShipped:
      payload:
        type: object
        required: [orderId, trackingNumber]
        properties:
          orderId: { type: string }
          trackingNumber: { type: string }
          carrier: { type: string }
          shippedAt: { type: string, format: date-time }
```

## Tooling Ecosystem

```bash
# OpenAPI
openapi-generator  # Code generation (50+ languages)
swagger-ui         # Interactive documentation
redoc              # Beautiful static docs
spectral           # Linting and validation
prism              # Mock server from spec
postman            # Import spec → create test collections

# AsyncAPI
asyncapi-generator # Code generation for event-driven systems
asyncapi-studio    # Visual editor
glee               # Testing and validation
```

## Why Solution Architects Must Acquire This

- **Spec-First Development**: Writing the OpenAPI spec BEFORE code forces API design thinking. Teams agree on the contract, then implement independently in parallel.
- **Single Source of Truth**: OpenAPI spec feeds: API gateway configuration, documentation portal, SDK generation, test generation, contract testing. One spec, many artifacts.
- **Security and Governance**: Spectral rules enforce API design standards: no `http://` URLs, all endpoints require auth, consistent error formats, standard pagination.

## Summary

| Specification | Domain | Current Version | Tools |
|--------------|--------|----------------|-------|
| OpenAPI 3.1 | Synchronous REST APIs | 3.1 | Swagger, OpenAPI Generator, Spectral |
| AsyncAPI 3.0 | Event-driven APIs | 3.0 | AsyncAPI Generator, Glee, Studio |
| JSON Schema | Data validation | 2020-12 | ajv, jsonschema |
| GraphQL Schema | GraphQL APIs | October 2021 | Apollo, GraphQL Codegen |

OpenAPI and AsyncAPI transform API contracts from documentation into code-level artifacts. An API designed spec-first has a machine-readable contract that drives code generation, validation, testing, and documentation. For architectures spanning REST and event-driven communication, OpenAPI + AsyncAPI together provide a complete specification of all system interfaces.
