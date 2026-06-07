# WebSocket

## Introduction
WebSocket is a protocol providing full-duplex communication over a single TCP connection, standardized by the IETF as RFC 6455 in 2011. Unlike HTTP's request-response model where the client always initiates, WebSocket enables the server to push data to the client at any time. This makes it fundamental for real-time applications: chat, live sports scores, collaborative editing, financial tickers, gaming, and live dashboards.

The WebSocket handshake starts as an HTTP upgrade request. Once established, the connection stays open, enabling bidirectional message passing with minimal overhead (2-6 bytes per frame vs hundreds of bytes for HTTP headers). This persistent connection model requires different architectural thinking than stateless HTTP—connection state, horizontal scaling with sticky sessions or pub/sub backplanes, and reconnection strategies.

## Definition

**WebSocket** is a computer communications protocol providing full-duplex communication channels over a single TCP connection. Key characteristics:

- **Full-duplex**: Both client and server can send messages independently
- **Persistent connection**: Single TCP connection reused for all messages
- **Low overhead**: 2-6 byte frame headers vs HTTP's verbose headers
- **Event-driven**: Messages arrive as events; no polling

## Concept Explanation

### The WebSocket Handshake

```
Client → Server (HTTP Upgrade):
  GET /ws HTTP/1.1
  Host: api.example.com
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13

Server → Client (101 Switching Protocols):
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### Server Implementation (Python)

```python
import asyncio
import websockets
import json
from typing import Dict, Set

class WebSocketServer:
    def __init__(self):
        self.connections: Dict[str, Set] = {}  # user_id → set of websocket connections
    
    async def handler(self, websocket, path):
        # Authenticate on connection
        user_id = await self._authenticate(websocket)
        if not user_id:
            await websocket.close(1008, "Authentication failed")
            return
        
        # Register connection
        if user_id not in self.connections:
            self.connections[user_id] = set()
        self.connections[user_id].add(websocket)
        
        try:
            # Heartbeat check
            asyncio.create_task(self._heartbeat(websocket, user_id))
            
            # Process messages
            async for message in websocket:
                await self._handle_message(user_id, json.loads(message))
        
        except websockets.exceptions.ConnectionClosed:
            pass
        finally:
            # Clean up on disconnect
            self.connections[user_id].discard(websocket)
            if not self.connections[user_id]:
                del self.connections[user_id]
    
    async def _heartbeat(self, websocket, user_id):
        """Send ping every 30s to detect dead connections"""
        while True:
            try:
                await asyncio.sleep(30)
                await websocket.ping()
            except websockets.exceptions.ConnectionClosed:
                break
    
    async def send_to_user(self, user_id: str, event: dict):
        """Send message to all connections for a user (mobile + desktop)"""
        if user_id in self.connections:
            message = json.dumps(event)
            for ws in self.connections[user_id]:
                try:
                    await ws.send(message)
                except:
                    pass

# Start server
start_server = websockets.serve(WebSocketServer().handler, "0.0.0.0", 8765)
asyncio.get_event_loop().run_until_complete(start_server)
asyncio.get_event_loop().run_forever()
```

### Client Implementation (JavaScript)

```javascript
class WebSocketClient {
    constructor(url) {
        this.url = url;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 10;
        this.connect();
    }
    
    connect() {
        this.ws = new WebSocket(this.url);
        
        this.ws.onopen = () => {
            console.log('Connected');
            this.reconnectAttempts = 0;
            // Authenticate
            this.send({ type: 'auth', token: getToken() });
        };
        
        this.ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            this.handleMessage(message);
        };
        
        this.ws.onclose = (event) => {
            if (!event.wasClean) {
                this.reconnect();
            }
        };
        
        this.ws.onerror = (error) => {
            console.error('WebSocket error:', error);
        };
    }
    
    reconnect() {
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
            this.reconnectAttempts++;
            setTimeout(() => this.connect(), delay);
        }
    }
    
    send(data) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(data));
        }
    }
    
    handleMessage(message) {
        switch (message.type) {
            case 'order_updated':
                this.onOrderUpdated(message.payload);
                break;
            case 'notification':
                this.onNotification(message.payload);
                break;
        }
    }
}
```

### Scaling WebSockets

```
PROBLEM: 100K concurrent connections = 100K open TCP connections
         Single server limit: ~65K connections per process (port exhaustion)

SOLUTIONS:

1. Horizontal Scaling + Pub/Sub Backplane:

   [Client A] ──→ [WS Server 1] ──┐
                                   ├── [Redis Pub/Sub] ── message broadcast
   [Client B] ──→ [WS Server 2] ──┘
   
   When WS Server 1 receives a message for user X connected to WS Server 2,
   it publishes to Redis; WS Server 2 subscribes and delivers to the client.

2. Sticky Sessions (simpler, but less resilient):
   Load balancer routes same user always to same server
   Server holds connection state locally (no pub/sub needed)
   
3. Managed services: AWS API Gateway WebSocket, Azure Web PubSub, 
   Cloud Run with session affinity
```

```python
# Redis Pub/Sub backplane for horizontal scaling
import redis.asyncio as redis

class PubSubBackplane:
    def __init__(self):
        self.redis = redis.Redis()
        self.pubsub = self.redis.pubsub()
        self.server_id = str(uuid.uuid4())
    
    async def subscribe(self):
        await self.pubsub.subscribe("websocket:messages")
        asyncio.create_task(self._listen())
    
    async def _listen(self):
        async for message in self.pubsub.listen():
            if message['type'] == 'message':
                data = json.loads(message['data'])
                if data['server_id'] != self.server_id:  # Ignore own messages
                    await self._deliver(data['user_id'], data['payload'])
    
    async def publish(self, user_id, payload):
        await self.redis.publish("websocket:messages", json.dumps({
            'server_id': self.server_id,
            'user_id': user_id,
            'payload': payload
        }))
```

### Reconnection and State Recovery

```javascript
// Client maintains sequence number for message ordering
let lastSequence = 0;

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    if (msg.sequence > lastSequence + 1) {
        // Gap detected—request missed messages
        ws.send(JSON.stringify({
            type: 'resync',
            fromSequence: lastSequence + 1
        }));
    }
    lastSequence = msg.sequence;
    handle(msg);
};

// On reconnect, server replays missed events
ws.onopen = () => {
    ws.send(JSON.stringify({
        type: 'auth',
        token: getToken(),
        lastSequence: lastSequence
    }));
};
```

## When to Use WebSocket vs Alternatives

| Technology | Best For | When NOT to Use |
|-----------|----------|-----------------|
| WebSocket | Bidirectional real-time, chat, gaming, collaboration | Simple server-to-client push (use SSE) |
| SSE (Server-Sent Events) | Server→client streaming, live feeds | Bidirectional needed, IE support |
| Long Polling | Fallback for old browsers | Modern applications (use WebSocket/SSE) |
| WebRTC | Peer-to-peer audio/video/data | Client-server communication |

## Summary

WebSocket provides true bidirectional, low-latency communication essential for real-time applications. The architectural challenge is scaling persistent connections horizontally (pub/sub backplane) and handling reconnection gracefully (exponential backoff + state recovery). For server-to-client-only push, SSE is simpler. For peer-to-peer real-time, WebRTC is appropriate. WebSocket is the correct choice when both client and server need to initiate messages independently with minimal overhead.
