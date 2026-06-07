# Databases & Storage

## Books
- **Database Internals** (Alex Petrov, 2019) — B-Trees, LSM-Trees, storage engines, distributed consensus. Deep dive into how databases work internally.
- **High Performance MySQL** (Baron Schwartz et al., 4th ed. 2021) — Query optimization, indexing, replication, scaling.
- **PostgreSQL: Up and Running** (Regina Obe, Leo Hsu) — Practical PostgreSQL administration and optimization.
- **The Data Warehouse Toolkit** (Ralph Kimball, 3rd ed. 2013) — Dimensional modeling, star schemas. The Kimball methodology bible.
- **Designing Data-Intensive Applications** (Kleppmann) — Chapters 3-4 on storage engines and encoding.

## Foundational Papers
- **A Comparison of Approaches to Large-Scale Data Analysis** (Pavlo et al., 2009) — MapReduce vs parallel databases benchmark
- **The Log-Structured Merge-Tree (LSM-Tree)** (O'Neil et al., 1996) — Foundation of RocksDB, LevelDB, Cassandra, Bigtable storage
- **Rethinking the Database for the Cloud Era** (Stonebraker, 2018) — Critique of traditional databases for cloud

## Key Concepts to Master
- B-Tree vs LSM-Tree trade-offs (read vs write optimization)
- MVCC implementations across PostgreSQL, MySQL, Oracle
- Row vs columnar storage and when to use each
- CAP theorem and real-world database classification
- Sharding strategies: range, hash, directory-based
- Index types: B-tree, Hash, GIN, GiST, BRIN, partial, covering
