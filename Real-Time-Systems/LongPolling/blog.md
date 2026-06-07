# Long Polling

Long Polling is the fallback real-time technique when WebSocket and SSE aren't available (legacy browsers, restrictive firewalls). The client sends an HTTP request; the server holds it open until data is available or a timeout occurs. On response (or timeout), the client immediately issues another request. To the client, it looks like near-real-time push.

It's less efficient than WebSocket or SSE (more overhead due to HTTP headers per cycle), but it works everywhere—any browser, any proxy, any firewall. Understanding it is important for architecting systems that must work in constrained environments.

## Server Implementation

```python
import time
import json
from flask import Flask, request

@app.route('/poll/orders')
def long_poll_orders():
    last_event_id = request.args.get('since', '0')
    timeout = 30  # seconds
    
    start = time.time()
    while time.time() - start < timeout:
        events = get_events_since(last_event_id)
        if events:
            return jsonify({'events': events, 'latest_id': events[-1]['id']})
        time.sleep(0.5)  # Check every 500ms
    
    # Timeout: return empty, client reconnects
    return jsonify({'events': [], 'latest_id': last_event_id})
```

## Client Implementation

```javascript
function longPoll(since) {
    fetch(`/poll/orders?since=${since}`)
        .then(res => res.json())
        .then(data => {
            data.events.forEach(handleEvent);
            longPoll(data.latest_id);  // Immediate reconnect
        })
        .catch(err => {
            setTimeout(() => longPoll(since), 5000);  // Retry after delay
        });
}

longPoll('0');  // Start polling
```

## Comparison with Other Real-Time Techniques

| Technique | Efficiency | Complexity | Universal Support | Use Case |
|-----------|-----------|------------|-------------------|----------|
| Long Polling | Low (HTTP overhead per cycle) | Low | Every browser, every proxy | Legacy fallback |
| SSE | Medium (persistent HTTP, no reconnect overhead) | Low | Modern browsers | Server→client streaming |
| WebSocket | High (minimal frame overhead) | Medium | Modern browsers, WebSocket-aware proxies | Bidirectional real-time |
| WebRTC | Highest (P2P media) | High | Modern browsers | Audio/video, P2P data |

## When Long Polling is Still Relevant

- **Enterprise environments** with restrictive proxies that block WebSocket upgrades but allow HTTP
- **Embedded systems** without EventSource API
- **Graceful degradation**: provide WebSocket + SSE as primary; Long Polling as fallback
- **Simple notification APIs** where the complexity of maintaining persistent connections isn't justified

Long Polling is the most universally compatible real-time technique. It works everywhere HTTP works—which is everywhere. Modern architectures should implement it as the fallback layer, with WebSocket or SSE as the primary transport.
