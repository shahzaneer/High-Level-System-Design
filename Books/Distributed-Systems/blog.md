# Distributed Systems

## Books
- **Distributed Systems** (Maarten van Steen, Andrew Tanenbaum, 4th ed. 2023) — Comprehensive textbook. Communication, naming, synchronization, consistency, fault tolerance.
- **Database Internals** (Alex Petrov) — Chapters on distributed consensus, replication, leader election.
- **Understanding Distributed Systems** (Roberto Vitillo, 2nd ed. 2022) — Practical introduction. Communication, coordination, scalability, resilience.

## Foundational Papers
- **Time, Clocks, and the Ordering of Events in a Distributed System** (Leslie Lamport, 1978) — Logical clocks, happened-before relation. The foundation of distributed systems theory.
- **The Byzantine Generals Problem** (Lamport, Shostak, Pease, 1982) — Fault tolerance in the presence of arbitrary failures.
- **Impossibility of Distributed Consensus with One Faulty Process** (Fischer, Lynch, Paterson, 1985) — The FLP impossibility result.
- **The Part-Time Parliament** (Lamport, 1998) — The Paxos consensus algorithm.
- **In Search of an Understandable Consensus Algorithm** (Ongaro, Ousterhout, 2014) — The Raft consensus algorithm. Designed to be understandable.
- **The Chubby Lock Service for Loosely-Coupled Distributed Systems** (Burrows, 2006) — Google's distributed lock service.
- **ZooKeeper: Wait-free coordination for Internet-scale systems** (Hunt et al., 2010) — Apache ZooKeeper design.

## Key Concepts
- Consensus (Paxos, Raft, Zab)
- Logical clocks, vector clocks, hybrid clocks
- Leader election, quorum, fencing
- Gossip protocols, failure detection
- CAP theorem, PACELC theorem
- Distributed transactions, 2PC, Saga
