# WebSocket Scaling

A single WebSocket server hits practical limits at ~50K-65K concurrent connections (port exhaustion, file descriptors, memory). Scaling beyond this requires horizontal scaling and a pub/sub backplane. The fundamental challenge: WebSocket connections are stateful and pinned to specific server instances. When Server A receives a message for a user connected to Server B, a mechanism must deliver it across instances.

## The Pub/Sub Backplane Pattern

```
[Client A] ──WS── [Server 1] ──┐
[Client B] ──WS── [Server 2] ──┤
[Client C] ──WS── [Server 3] ──┤
                               ├── [Redis Pub/Sub] or [Kafka]
[Client D] ──WS── [Server 1] ──┤
[Client E] ──WS── [Server 2] ──┘

When Server 1 needs to send to Client E (connected to Server 2):
  Server 1 → Redis publish("ws:msg", {user_id: "E", payload: ...})
  Server 2 → Redis subscribe → receives → delivers to Client E's connection
```

## Implementation

```python
class ScalableWSServer:
    def __init__(self):
        self.server_id = str(uuid.uuid4())
        self.local_connections = {}  # user_id → websocket
        self.redis = redis.Redis()
    
    async def handle_client(self, websocket, user_id):
        self.local_connections[user_id] = websocket
        try:
            async for message in websocket:
                # Process locally, broadcast via Redis
                await self.broadcast_to_user(message['to'], message)
        finally:
            del self.local_connections[user_id]
    
    async def broadcast_to_user(self, user_id, message):
        # Check if user is connected to THIS instance
        if user_id in self.local_connections:
            await self.local_connections[user_id].send(message)
        else:
            # Publish to Redis—other servers will pick it up
            self.redis.publish('ws:messages', json.dumps({
                'server_id': self.server_id,
                'user_id': user_id,
                'message': message
            }))
    
    async def redis_listener(self):
        pubsub = self.redis.pubsub()
        pubsub.subscribe('ws:messages')
        async for msg in pubsub.listen():
            if msg['type'] == 'message':
                data = json.loads(msg['data'])
                # Ignore messages from my own server (already delivered locally)
                if data['server_id'] != self.server_id:
                    if data['user_id'] in self.local_connections:
                        await self.local_connections[data['user_id']].send(data['message'])
```

## Connection Drain and Graceful Shutdown

```python
import signal

class GracefulShutdown:
    def __init__(self):
        self.draining = False
    
    def start_drain(self):
        """Called before server shutdown"""
        self.draining = True
        # Stop accepting new connections (remove from load balancer)
        # Wait for existing connections to complete
        for ws in self.local_connections.values():
            ws.send(json.dumps({
                'type': 'server_shutdown',
                'reconnect_in': 5,
                'new_endpoint': 'wss://other-server/ws'
            }))
        # Give clients 5 seconds to reconnect to other server
        time.sleep(5)
```

## Load Balancer Considerations

```nginx
# Nginx WebSocket proxy
upstream ws_backend {
    ip_hash;  # Sticky sessions: same client → same server
    server ws1:8080;
    server ws2:8080;
}

server {
    location /ws {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_read_timeout 3600s;
    }
}
```

## Capacity Planning

```
1 WebSocket connection ≈ 5-10 KB memory (kernel buffers + app state)
100K connections ≈ 500 MB - 1 GB memory
1M connections ≈ 5-10 GB memory

Network: 100K idle connections ≈ 0 bandwidth (TCP keepalives: ~1 packet/min)

Key metric: connections per server instance. Monitor:
  - Active connections count
  - Memory per connection
  - Message throughput (messages/sec per instance)
  - Redis pub/sub message rate
```

## Summary

Scaling WebSockets requires a pub/sub backplane (Redis, Kafka) for cross-instance message delivery and sticky sessions at the load balancer. The architecture uses Redis as a message bus: each server subscribes to all messages, but only delivers to users connected locally. This enables linear horizontal scaling—add more servers to handle more connections.
