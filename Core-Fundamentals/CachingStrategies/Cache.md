# Caching Strategies

### Why fast systems are usually smart, not powerful

Your database is slow.
Not because it’s bad.
Because it’s doing exactly what you asked it to do… too often.

Every high-performance system you admire relies on **caching**. Not as an optimization, but as a **core architectural layer**.

---

## What Is Caching?

Caching is the practice of **storing frequently accessed data closer to the consumer** so future requests can be served faster.

Instead of:
Client → App → Database

You get:
Client → Cache → App → Database (only on cache miss)

Less latency. Less load. Happier systems.

---

## Why Caching Is Needed

Caching exists to solve real problems:

* Reduce database load
* Improve response times
* Handle traffic spikes
* Lower infrastructure costs
* Improve system resilience

Without caching, horizontal scaling becomes expensive and vertical scaling hits limits quickly.

## Cache Eviction Strategies

Cache is limited. Something must go.

### Common Eviction Policies

* **LRU (Least Recently Used)**
  Removes data not accessed recently

* **LFU (Least Frequently Used)**
  Removes data used least often

* **FIFO (First In, First Out)**
  Simple but dumb

* **TTL (Time To Live)**
  Data expires after time

**Architect tip:**
TTL + LRU is the most practical combo.

---

## Cache Consistency Strategies

Caching is easy.
Keeping it correct is hard.

### Techniques:

* TTL-based expiration
* Explicit cache invalidation
* Versioned keys
* Event-driven invalidation

**Truth:**
Perfect cache consistency is expensive. Most systems choose “good enough.”

---

## Caching and Scaling Decisions

### Vertical Scaling + Caching

* In-memory cache on a single server
* Fast but fragile
* Single point of failure

### Horizontal Scaling + Caching

* Distributed cache (Redis cluster)
* Shared across services
* Highly available and scalable

**Rule:**
If your app scales horizontally, your cache must too.

---

## Common Caching Pitfalls (Avoid These)

* Caching everything blindly
* Forgetting cache invalidation
* Using cache as a database
* No monitoring on cache hit ratio
* Ignoring cold-start behavior

Bad caching is worse than no caching.

---

## Real-World Examples

* **Netflix:** CDN + edge caching
* **Instagram:** Feed caching with TTL
* **E-commerce:** Product catalog cache, inventory carefully handled
* **Auth systems:** Token and session caching

---

## Architect’s Final Take

Caching is not an optimization.
It is a **fundamental architectural decision**.

Good caching:

* Makes systems fast
* Reduces cost
* Improves reliability

Bad caching:

* Creates bugs
* Breaks consistency
* Makes failures harder to debug

Design your cache like you design your database. With intention.

