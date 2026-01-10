## Session Persistence (Sticky Sessions)
## What Is It?
- Session persistence ensures that a user's subsequent requests go to the same backend server. Necessary when application stores session state locally on the server (not in a shared session store).
## Methods:
1. Source IP Affinity:

Uses client IP address (similar to IP Hash)
L4 or L7
Problem: Users behind NAT/proxies appear as same IP

2. Cookie-Based Persistence:

Load balancer inserts a cookie with server identifier L7 only
Most reliable method
Example: Set-Cookie: AWSALB=server-3; Path=/; HttpOnly

3. Session ID-Based:

Extracts session ID from cookie/URL, hashes to select server
L7 only
More intelligent than pure IP-based

4. SSL Session ID:

Uses TLS session ID for persistence
L4 (but SSL-aware)
Useful when cookies aren't available

## Problems with Sticky Sessions:

Uneven Load: Popular users' sessions all on one server
Failover Issues: If server dies, all sessions lost
Scaling Difficulty: Can't remove servers without disrupting sessions
### Modern Best Practice: Use stateless sessions (JWT, shared session stores like Redis) and avoid sticky sessions when possible