# gRPC

## Introduction
gRPC (gRPC Remote Procedure Call) is a high-performance, open-source RPC framework developed by Google. It uses Protocol Buffers (protobuf) as its interface definition language and binary serialization format, and HTTP/2 as its transport. gRPC is designed for low-latency, high-throughput communication between services—the sweet spot for microservices architectures where services call each other thousands of times per second.

Unlike REST (text-based JSON over HTTP/1.1), gRPC is binary, multiplexed, and supports bidirectional streaming out of the box. It generates client and server code in 10+ languages from a single `.proto` definition. For server-to-server communication within a data center or cluster, gRPC offers 7-10x better performance than REST/JSON. It is the standard for service-to-service communication at Google, Netflix, Square, and Cisco.

## Definition

**gRPC** is a modern RPC framework that enables client and server applications to communicate transparently and efficiently, using Protocol Buffers for service definition and binary serialization, and HTTP/2 for transport with support for request-response, server-streaming, client-streaming, and bidirectional streaming.

**Key concepts**:
- **Service**: Defined in .proto file with RPC methods
- **Message**: Structured data defined as protobuf messages
- **Stub**: Auto-generated client code that makes RPC calls look like local function calls
- **Channel**: A virtual connection to a service endpoint, supporting load balancing and retry
- **Streaming**: Server-streaming, client-streaming, bidirectional-streaming

## Concept Explanation

### Protocol Buffers (.proto)

```protobuf
syntax = "proto3";

package orders;

service OrderService {
  // Unary: simple request-response
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse);
  
  // Server streaming: client sends one request, receives stream of responses
  rpc ListOrders (ListOrdersRequest) returns (stream Order);
  
  // Client streaming: client sends stream, server responds once
  rpc BulkCreateOrders (stream CreateOrderRequest) returns (BulkCreateResponse);
  
  // Bidirectional streaming: both sides send streams
  rpc ProcessOrders (stream OrderUpdate) returns (stream OrderStatus);
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
}

message OrderItem {
  string sku = 1;
  int32 quantity = 2;
  double unit_price = 3;
}

message CreateOrderResponse {
  string order_id = 1;
  OrderStatus status = 2;
}

message Order {
  string order_id = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  double total = 4;
  OrderStatus status = 5;
  int64 created_at = 6;  // Unix timestamp
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}
```

### Field Numbers and Backward Compatibility

```
Protobuf field numbers are the wire-format identifiers:
  customer_id = 1  → on the wire, this is field #1

Rules for backward-compatible changes:
  ✓ Add new fields (with new field numbers)
  ✓ Mark fields as deprecated
  ✗ Never rename field numbers
  ✗ Never change field types
  ✗ Never remove required fields (proto2) or change type of existing fields

This is safer than JSON where renaming "customerId" breaks all consumers.
```

### Server Implementation (Python)

```python
import grpc
from concurrent import futures
import orders_pb2, orders_pb2_grpc

class OrderService(orders_pb2_grpc.OrderServiceServicer):
    
    def CreateOrder(self, request, context):
        # Unary: receive request, return response
        order = self._create_order(request)
        
        # Set status code and metadata
        context.set_code(grpc.StatusCode.OK)
        context.set_trailing_metadata([('order-id', order.order_id)])
        
        return orders_pb2.CreateOrderResponse(
            order_id=order.order_id,
            status=order.status
        )
    
    def ListOrders(self, request, context):
        # Server streaming: yield responses
        orders = db.query("SELECT * FROM orders WHERE customer_id = ?", 
                          [request.customer_id])
        for order in orders:
            yield self._order_to_proto(order)
    
    def BulkCreateOrders(self, request_iterator, context):
        # Client streaming: iterate over incoming stream
        created_count = 0
        for request in request_iterator:
            self._create_order(request)
            created_count += 1
        return orders_pb2.BulkCreateResponse(created=created_count)
    
    def ProcessOrders(self, request_iterator, context):
        # Bidi streaming: both sides stream
        for request in request_iterator:
            # Process each incoming update
            status = self._process_update(request)
            # Send response back on same stream
            yield orders_pb2.OrderStatus(order_id=request.order_id, status=status)

# Start server
server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
orders_pb2_grpc.add_OrderServiceServicer_to_server(OrderService(), server)
server.add_insecure_port('[::]:50051')
server.start()
```

### Client Implementation

```python
import grpc
import orders_pb2, orders_pb2_grpc

# Create channel and stub
channel = grpc.insecure_channel('order-service:50051')
stub = orders_pb2_grpc.OrderServiceStub(channel)

# Unary call
response = stub.CreateOrder(orders_pb2.CreateOrderRequest(
    customer_id="CUST-789",
    items=[
        orders_pb2.OrderItem(sku="SKU-1", quantity=2, unit_price=29.99)
    ]
))
print(f"Created order: {response.order_id}")

# Server streaming
for order in stub.ListOrders(orders_pb2.ListOrdersRequest(customer_id="CUST-789")):
    print(f"Order: {order.order_id}, Status: {order.status}")

# With deadline/timeout
try:
    response = stub.CreateOrder(request, timeout=5)  # 5 second deadline
except grpc.RpcError as e:
    if e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
        print("Request timed out")
```

### Load Balancing Options

```yaml
# gRPC load balancing strategies:

# 1. Proxy LB (simplest): Envoy/Nginx in front of gRPC servers
#    Client → LB → gRPC servers

# 2. Client-side LB with DNS: Client resolves DNS → gets all server IPs → 
#    Client picks one and maintains long-lived HTTP/2 connection

# 3. Client-side LB with xDS (Envoy/Service Mesh):
#    Lookaside load balancing with service discovery
```

### gRPC vs REST Comparison

| Feature | gRPC | REST |
|---------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 (or HTTP/2) |
| Payload format | Protobuf (binary) | JSON (text) |
| Payload size | 3-10x smaller | Human-readable |
| Streaming | Bidirectional native | Requires WebSocket/SSE |
| Code generation | Native (protoc) | OpenAPI Generator |
| Browser support | Limited (gRPC-Web) | Universal |
| Debugging | Harder (binary, needs tools) | Easy (curl, browser) |
| Performance | 7-10x faster | Baseline |

## When to Use gRPC

```
Use gRPC for:
  ✓ Microservice-to-microservice communication
  ✓ Low-latency, high-throughput internal APIs
  ✓ Streaming (real-time data feeds, event streams)
  ✓ Polyglot environments (codegen in 10+ languages)
  ✓ Mobile clients (binary = smaller payloads = less battery/data)

Use REST for:
  ✓ Public APIs consumed by third parties
  ✓ Web browser clients (native JSON support)
  ✓ Caching at CDN level
  ✓ Debugging and exploration (curl, browser)

Use GraphQL for:
  ✓ Flexible client-driven data fetching
  ✓ Aggregating multiple backends
  ✓ Mobile apps with varying data needs
```

## Summary

gRPC is the high-performance choice for service-to-service communication. Protobuf's binary serialization and strong typing provide both speed and safety. Server and client code generation eliminates boilerplate and ensures compatibility. For internal microservices, gRPC should be the default choice—REST is for external APIs. The combination of gRPC internally + REST/GraphQL externally (via API gateway) is the modern architectural pattern for most distributed systems.
