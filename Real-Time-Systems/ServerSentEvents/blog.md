# Server-Sent Events (SSE)

SSE is a simple, HTTP-based protocol for server-to-client streaming. The server holds an HTTP connection open and pushes events as they occur. Unlike WebSocket (bidirectional), SSE is unidirectional—server to client only. The client uses the browser's EventSource API; no custom client library needed. SSE is simpler than WebSocket for one-way data flow: live dashboards, notification streams, log tailing, and progress updates.

SSE's key advantage: it works over standard HTTP, meaning it benefits from HTTP/2 multiplexing, existing load balancers, and HTTP authentication. Unlike WebSocket, SSE automatically reconnects when the connection drops.

## Server Implementation (Python/Flask)

```python
import json
import time
from flask import Flask, Response, stream_with_context

@app.route('/events/orders')
def order_events():
    def generate():
        last_event_id = request.headers.get('Last-Event-ID')
        # Replay missed events if client reconnected
        if last_event_id:
            events = get_events_since(last_event_id)
            for event in events:
                yield format_sse_event(event)
        
        # Stream new events
        for event in order_event_stream():
            yield format_sse_event(event)
    
    return Response(
        stream_with_context(generate()),
        mimetype='text/event-stream',
        headers={
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
            'X-Accel-Buffering': 'no',  # Disable nginx buffering
        }
    )

def format_sse_event(event):
    return f"""id: {event['id']}
event: {event['type']}
data: {json.dumps(event['payload'])}
retry: 3000

"""
```

## Client Implementation

```javascript
const source = new EventSource('/events/orders');

source.addEventListener('OrderPlaced', (e) => {
    const order = JSON.parse(e.data);
    updateDashboard(order);
});

source.addEventListener('OrderShipped', (e) => {
    const order = JSON.parse(e.data);
    notifyCustomer(order);
});

source.onerror = (e) => {
    // EventSource auto-reconnects; no manual reconnect needed
    console.log('Connection lost, auto-reconnecting...');
};
```

## SSE vs WebSocket

| Feature | SSE | WebSocket |
|---------|-----|-----------|
| Direction | Server → Client only | Bidirectional |
| Protocol | HTTP | WebSocket (upgrade from HTTP) |
| Auto-reconnect | Built-in (EventSource) | Must implement manually |
| Binary data | No (text only) | Yes |
| HTTP/2 | Native multiplexing | Requires separate connections |
| Load balancer | Works with standard HTTP LBs | Requires WebSocket-aware LBs |

## Scaling SSE

```python
# Redis pub/sub to broadcast events across server instances
# Each server subscribes to Redis; pushes to its connected clients

class SSEServer:
    def __init__(self):
        self.clients = {}  # user_id → [response_objects]
        self.redis = redis.Redis()
    
    def broadcast(self, event):
        self.redis.publish('sse:events', json.dumps(event))
    
    def _redis_listener(self):
        pubsub = self.redis.pubsub()
        pubsub.subscribe('sse:events')
        for message in pubsub.listen():
            if message['type'] == 'message':
                event = json.loads(message['data'])
                # Deliver to locally connected clients
                for client in self.clients.get(event['user_id'], []):
                    client.send(format_sse_event(event))
```

SSE is the underrated workhorse of real-time data delivery. For any use case where the server pushes data to clients and client-to-server communication uses regular REST, SSE is simpler, more robust, and more infrastructure-friendly than WebSocket.
