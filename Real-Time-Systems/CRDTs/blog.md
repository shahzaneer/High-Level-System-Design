# CRDTs (Conflict-Free Replicated Data Types)

CRDTs are data structures that enable multiple users to edit the same data concurrently without conflicts—ideal for collaborative editing (Google Docs, Figma), distributed counters, and multi-leader database replication. The key property: CRDTs are mathematically guaranteed to converge to the same state across all replicas, regardless of the order in which operations are applied.

CRDTs eliminate the need for conflict resolution in multi-master replication. Traditional multi-master databases face "last-write-wins" (LWW) semantics or require manual conflict resolution. CRDTs provide automatic, deterministic convergence without coordination—a property that enables offline-first and real-time collaborative applications.

## Key CRDT Types

| CRDT | Use Case | Merge Operation |
|------|----------|----------------|
| G-Counter (Grow-only) | Page views, likes | max(values) |
| PN-Counter (Positive-Negative) | Inventory, upvotes/downvotes | increments - decrements |
| G-Set (Grow-only Set) | Unique visitors, tags | union(sets) |
| 2P-Set (Two-Phase Set) | Todo lists | add_set - remove_set |
| LWW-Register (Last-Write-Wins) | User profile fields | timestamp comparison |
| OR-Set (Observed-Remove Set) | Shopping cart | add_set with unique IDs |
| RGA / WOOT / Yjs | Collaborative text editing | Insert operations with positions |

## G-Counter (Grow-Only Counter)

```python
class GCounter:
    """
    Each replica maintains a map of replica_id → count.
    Value = sum of all counts. Merge = element-wise max.
    """
    
    def __init__(self, replica_id):
        self.replica_id = replica_id
        self.state = {}  # {replica_id: count}
    
    def increment(self, amount=1):
        self.state[self.replica_id] = self.state.get(self.replica_id, 0) + amount
    
    def value(self):
        return sum(self.state.values())
    
    def merge(self, other_state):
        """Merge another replica's state—commutative, associative, idempotent"""
        for replica, count in other_state.items():
            self.state[replica] = max(self.state.get(replica, 0), count)
    
    def to_dict(self):
        return dict(self.state)

# Replica A: increment 5, increment 3
a = GCounter('A')
a.increment(5)
a.increment(3)  # A: {A: 8}

# Replica B: increment 2
b = GCounter('B')
b.increment(2)  # B: {B: 2}

# Merge (any order yields same result)
a.merge(b.state)  # A: {A: 8, B: 2} → value = 10
b.merge(a.state)  # B: {A: 8, B: 2} → value = 10  SAME RESULT
```

## PN-Counter (Positive-Negative Counter)

```python
class PNCounter:
    """
    Supports increment AND decrement.
    Uses two G-Counters: one for increments, one for decrements.
    Value = sum(inc) - sum(dec)
    """
    
    def __init__(self, replica_id):
        self.inc = GCounter(replica_id)
        self.dec = GCounter(replica_id)
    
    def increment(self, amount=1):
        self.inc.increment(amount)
    
    def decrement(self, amount=1):
        self.dec.increment(amount)
    
    def value(self):
        return self.inc.value() - self.dec.value()
    
    def merge(self, other):
        self.inc.merge(other['inc'])
        self.dec.merge(other['dec'])
```

## Collaborative Text Editing (RGA/Yjs Approach)

```python
# Simplified RGA (Replicated Growable Array) for text editing
# Each character gets a globally unique position ID: [sequence_number, replica_id]
# Characters are ordered by position ID.
# Concurrent inserts at same position: tiebreaker by replica_id

class RGAText:
    def __init__(self, replica_id):
        self.replica_id = replica_id
        self.seq = 0
        self.chars = {}  # position_id → character
    
    def insert(self, char, after_position=None):
        self.seq += 1
        position = [self.seq, self.replica_id]
        self.chars[tuple(position)] = char
        return position
    
    def get_text(self):
        return ''.join(
            char for pos, char in sorted(self.chars.items(), 
                                         key=lambda x: (x[0][0], x[0][1]))
        )
    
    def merge(self, other_chars):
        for pos, char in other_chars.items():
            if pos not in self.chars:
                self.chars[pos] = char

# Why this converges:
# - Every insert gets a unique position (sequence_number, replica_id)
# - Position IDs are totally ordered
# - Merge = union of all inserts
# - No deletes in this simplified version (OR-set needed for deletes)
```

## When CRDTs Are the Answer

```
USE CRDTs for:
  ✓ Collaborative editing (Google Docs, Notion, Figma)
  ✓ Offline-first applications (mobile apps that sync when reconnected)
  ✓ Multi-leader database replication without conflict resolution
  ✓ Distributed counters (page views, analytics across regions)

DON'T USE CRDTs for:
  ✗ Strongly consistent systems requiring immediate agreement
  ✗ Simple key-value stores (LWW-Register is sufficient)
  ✗ Systems where conflict resolution is acceptable (application-level merge functions)
```

## Production Libraries

- **Yjs**: CRDT framework for collaborative editing (used by Linear, Room.sh)
- **Automerge**: JSON-like CRDT library
- **Riak DT**: CRDTs built into Riak KV
- **Redis CRDT**: Enterprise Redis supports CRDT-based multi-master replication

CRDTs solve the fundamental challenge of concurrent data modification without coordination. They are the mathematical foundation of collaborative software and multi-leader replication. The trade-off: CRDTs are more complex than last-write-wins and consume more storage (metadata per operation), but they provide deterministic convergence that LWW cannot.
