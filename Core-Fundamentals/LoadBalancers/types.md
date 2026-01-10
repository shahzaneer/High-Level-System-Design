# OSI Layer Classification

- Understanding which OSI layer a load balancer operates at is **crucial** because it determines **what information the load balancer can use for routing decisions** and **what protocols it can handle**.

---

## Layer 4 (Transport Layer) Load Balancers

### What It Is
- Layer 4 load balancers operate at the **Transport layer (TCP/UDP)**.
- They make routing decisions based on **network-level information** available in TCP or UDP headers:
  - `Source IP`
  - `Destination IP`
  - `Source port`
  - `Destination port`
  - `Protocol type`
- They **do NOT inspect packet content**.

### How It Works
1. When a client initiates a TCP connection, the L4 load balancer performs **NAT (Network Address Translation)**.
2. Client connects to load balancer VIP: `203.0.113.1:80`
3. Load balancer selects backend server: `10.0.1.5:8080`
4. Packets are rewritten:
   - Destination changes from VIP to backend IP
5. Backend server responds to load balancer
6. Load balancer rewrites source IP back to VIP
7. Client sees response from the original VIP

The connection appears **direct** to both client and server, but the load balancer is silently routing packets in between.

### Advantages
- **Very Fast**: Packet-level routing, extremely high throughput, millions of connections per second
- **Protocol Agnostic**: Works with any TCP or UDP protocol  
  `HTTP, HTTPS, SMTP, FTP, databases, custom protocols`
- **Lower Latency**: No deep packet inspection
- **Less Resource Intensive**: Minimal CPU and memory usage

### Disadvantages
- **Limited Routing Logic**: No URL, header, or cookie inspection
- **No SSL Intelligence**: Can pass encrypted traffic but cannot inspect it
- **Basic Session Persistence**: IP-based or port-based only
- **No Content Caching**

### Use Cases
- Non-HTTP protocols such as `databases, message queues, gaming servers`
- **Maximum performance** requirements
- Simple `round-robin` traffic distribution
- When SSL/TLS must remain **end-to-end encrypted**

---

## Layer 7 (Application Layer) Load Balancers

### What It Is
- Layer 7 load balancers operate at the **Application layer (HTTP/HTTPS)**.
- They fully understand the application protocol and act as a **full reverse proxy**.
- Capabilities include:
  - SSL decryption
  - Reading HTTP headers
  - Inspecting URLs
  - Parsing cookies
  - Examining request bodies

### How It Works
The L7 load balancer **terminates the client connection** and creates a new one to the backend.

1. Client establishes TCP connection and SSL handshake with load balancer
2. Load balancer decrypts HTTPS and reads the HTTP request
3. Routing decision based on:
   - `URL path`
   - `Headers`
   - `Cookies`
   - `HTTP methods`
4. Load balancer opens a **separate TCP connection** to backend
5. Request may be modified before forwarding
6. Backend responds to load balancer
7. Load balancer may:
   - Cache
   - Compress
   - Modify response
8. Response sent back to client

This creates **two separate connections**:
- `Client ↔ Load Balancer`
- `Load Balancer ↔ Backend`

### Advantages
- **Intelligent Routing**: URL paths, headers, cookies, request methods
- **Content-Based Decisions**:
  - `/api/*` → API servers
  - `/images/*` → media servers
- **SSL Termination**: Reduces backend CPU usage
- **Advanced Session Persistence**: Cookie-based sticky sessions
- **Request Manipulation**: Header injection, URL rewriting, compression
- **Security Features**: `WAF`, rate limiting, authentication
- **Caching**: Reduces backend load

### Disadvantages
- **Higher Latency**: Full protocol processing adds milliseconds
- **More Resource Intensive**: Higher CPU and memory usage
- **Lower Throughput**: Fewer connections per second than L4
- **Protocol Specific**: Primarily HTTP and HTTPS
- **Configuration Complexity**: More knobs, more chances to misconfigure

### Use Cases
- Web applications needing **intelligent routing**
- Microservices with `path-based routing`
- SSL offloading requirements
- Scenarios where caching improves performance
- Advanced security needs like `WAF`, authentication, rate limiting

---

## Layer 3 (Network Layer) Load Balancers

### What It Is
- Operates at the **IP layer**.
- Uses techniques such as:
  - `Direct Server Return (DSR)`
  - `IP-over-IP tunneling`
- Less common than L4 and L7.

### How It Works
- Uses routing and IP manipulation.
- In **DSR mode**:
  - Load balancer forwards incoming packets to backend servers
  - Backend servers respond **directly to the client**
  - Response traffic bypasses the load balancer entirely

This dramatically improves performance and scalability.

### Use Cases
- **Ultra-high performance** environments
- Scenarios where return traffic would overwhelm the load balancer
- Specialized and advanced networking architectures
