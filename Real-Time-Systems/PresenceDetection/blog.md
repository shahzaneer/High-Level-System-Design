# Presence Detection

Presence Detection tracks which users are currently online/connected. It's fundamental to chat applications, collaboration tools, and multiplayer games. The architectural challenge: presence is a soft-state problem. Users disconnect without notice (browser close, network loss, laptop sleep), and the system must detect this and update presence state within seconds.

## Heartbeat-Based Detection

```
Client sends heartbeat every 30s
Server tracks last_heartbeat timestamp
Server periodically scans for users with last_heartbeat > 90s ago → mark offline
```

```python
import asyncio
import time

class PresenceTracker:
    def __init__(self, redis):
        self.redis = redis
        self.local_heartbeats = {}  # user_id → last_heartbeat_timestamp
    
    async def record_heartbeat(self, user_id):
        now = time.time()
        self.local_heartbeats[user_id] = now
        # Persist to Redis with TTL—if TTL expires, key disappears
        await self.redis.setex(f"presence:{user_id}", 120, json.dumps({
            'user_id': user_id,
            'last_seen': now,
            'status': 'online'
        }))
    
    async def check_presence(self, user_id):
        data = await self.redis.get(f"presence:{user_id}")
        if data:
            return json.loads(data)
        return {'user_id': user_id, 'status': 'offline'}
    
    async def get_online_users(self, user_ids):
        pipe = self.redis.pipeline()
        for uid in user_ids:
            pipe.exists(f"presence:{uid}")
        results = pipe.execute()
        return [uid for uid, exists in zip(user_ids, results) if exists]

# Background task: sweep for stale local connections
async def stale_connection_sweeper(presence, timeout=90):
    while True:
        now = time.time()
        stale = [uid for uid, ts in presence.local_heartbeats.items() 
                 if now - ts > timeout]
        for uid in stale:
            await presence.redis.delete(f"presence:{uid}")
            del presence.local_heartbeats[uid]
        await asyncio.sleep(30)
```

## WebSocket Disconnect Detection

```python
# WebSocket close event = immediate offline update
async def handle_disconnect(self, websocket, user_id):
    await self.redis.delete(f"presence:{user_id}")
    # Broadcast presence change to friends/subscribers
    await self.broadcast_presence_change(user_id, 'offline')

# But what about browser crashes? TCP doesn't close cleanly.
# Solution: server-side ping/pong
async def heartbeat_monitor(self, websocket, user_id):
    try:
        while True:
            await asyncio.sleep(30)
            await websocket.ping()
            # If ping fails (connection dead), exception triggers disconnect handler
    except:
        await self.handle_disconnect(websocket, user_id)
```

## Presence at Scale

```
For millions of users, Redis SET for every heartbeat is expensive.
Optimization: Use Redis Sorted Sets for online users.

ZADD presence:users {timestamp} {user_id}     -- Add/update user
ZREMRANGEBYSCORE presence:users 0 {now-90}    -- Remove stale (>90s old)
ZRANGE presence:users 0 -1                     -- Get all online users

Single Redis operation sweeps stale users (instead of per-user TTL check)
```

## Broadcast Presence Changes

```python
# When user goes online/offline, notify their contacts
async def broadcast_presence_change(self, user_id, status):
    # Get user's contacts/friends
    contacts = await self.get_contacts(user_id)
    
    for contact_id in contacts:
        # Publish to Redis; each server delivers to locally connected contacts
        await self.redis.publish('presence:updates', json.dumps({
            'user_id': user_id,
            'status': status,
            'notify': contact_id
        }))
```

## Data Model

```sql
-- Persisted presence state (for "last seen" timestamps)
CREATE TABLE user_presence (
    user_id UUID PRIMARY KEY,
    status VARCHAR(10),        -- online, offline, away
    last_seen TIMESTAMP,
    last_online TIMESTAMP
);

-- Redis (ephemeral, fast):
-- key: presence:{user_id} TTL: 120s
-- value: {"status":"online","last_seen":1700000000,"device":"mobile"}
```

Presence is a soft-state problem best solved with ephemeral storage (Redis) + heartbeats. The key insight: heartbeat TTL creates automatic cleanup—if a user disappears, their presence key expires naturally without explicit "disconnect" handling.
