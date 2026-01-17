### **Technical Leadership Decisions**

**Choosing Partitioning vs Sharding**:

| Metric      | Threshold  | Solution                |
| ----------- | ---------- | ----------------------- |
| Table rows  | < 100M     | No partitioning needed  |
| Table rows  | 100M - 1B  | Partitioning            |
| Table rows  | > 1B       | Partitioning + Sharding |
| Queries/sec | < 10K      | Single database         |
| Queries/sec | 10K - 50K  | Read replicas           |
| Queries/sec | > 50K      | Sharding required       |
| Storage     | < 1TB      | Single database         |
| Storage     | 1TB - 10TB | Partitioning            |
| Storage     | > 10TB     | Sharding                |

---

## **Summary: Decision Matrix**

| Scenario                     | Solution                                | Reason                                 |
| ---------------------------- | --------------------------------------- | -------------------------------------- |
| 100M rows, single region     | **Partitioning** (PostgreSQL/MySQL)     | No need for separate servers yet       |
| 1B rows, single region       | **Partitioning + Read Replicas**        | Scale reads, partition for query speed |
| 10B rows, global users       | **Sharding** (geographic)               | Latency + scale requirements           |
| Unpredictable growth         | **Managed sharding** (DynamoDB, Cosmos) | Auto-scaling, ops-free                 |
| Strong consistency required  | **Cloud Spanner** or **2PC sharding**   | Global transactions                    |
| Eventual consistency OK      | **Cassandra** or **DynamoDB**           | High availability, AP system           |
| Time-series data             | **Range partitioning** by timestamp     | Easy archival, query optimization      |
| User data, even distribution | **Hash sharding**                       | Uniform load                           |
| Multi-tenant SaaS            | **Shard by tenant_id**                  | Isolation, compliance                  |

**Golden Rule**: Start simple (single database) → Add read replicas → Partition large tables → Shard when you must (> 50K QPS or > 10TB).
