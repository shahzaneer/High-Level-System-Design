# Core System Design

## Must-Read Books
- **Designing Data-Intensive Applications** (Martin Kleppmann, 2017) — *The* system design bible. Covers replication, partitioning, transactions, consensus, batch/stream processing.
- **Building Microservices** (Sam Newman, 2nd ed. 2021) — Microservice decomposition, integration patterns, testing, deployment.
- **System Design Interview** (Alex Xu, 2020) — Practical walkthroughs of designing YouTube, Twitter, WhatsApp, etc.
- **Fundamentals of Software Architecture** (Mark Richards, Neal Ford, 2020) — Architecture patterns, soft skills, architecture characteristics.
- **Software Architecture: The Hard Parts** (Neal Ford et al., 2021) — Trade-offs in distributed systems, data decomposition, sagas.

## Foundational Papers
- **Dynamo: Amazon's Highly Available Key-value Store** (2007) — Eventual consistency, consistent hashing, hinted handoff. The foundation of DynamoDB, Cassandra, Riak.
- **Bigtable: A Distributed Storage System for Structured Data** (2006) — Column-oriented storage. Foundation of HBase, Bigtable.
- **MapReduce: Simplified Data Processing on Large Clusters** (2004) — Batch processing paradigm. Foundation of Hadoop, Spark.
- **The Google File System** (2003) — Distributed file system. Foundation of HDFS.
- **Dapper: Large-Scale Distributed Systems Tracing** (2010) — Distributed tracing. Foundation of Zipkin, Jaeger.
- **Spanner: Google's Globally-Distributed Database** (2012) — TrueTime, external consistency. Foundation of Cloud Spanner, CockroachDB.
- **TAO: Facebook's Distributed Data Store** (2013) — Graph-based social network data model.
- **Kafka: A Distributed Messaging System for Log Processing** (2011) — Log-based messaging. Foundation of Apache Kafka.

## Blogs & Resources
- **High Scalability** (highscalability.com) — Real-world architecture case studies
- **Netflix Tech Blog** — Microservices, chaos engineering, CDN at scale
- **Uber Engineering Blog** — Domain-oriented microservices, real-time systems
- **AWS Architecture Blog** — Cloud-native architecture patterns
- **Martin Fowler's Blog** (martinfowler.com) — Patterns, microservices, evolutionary architecture

## System Design Interview Prep
- **Grokking the System Design Interview** — Structured approach to design questions
- **System Design Primer** (GitHub: donnemartin/system-design-primer) — Comprehensive study guide
- **ByteByteGo** (Alex Xu) — Visual explanations of complex systems
- Practice: Design a URL shortener, chat system, news feed, rate limiter, distributed message queue
