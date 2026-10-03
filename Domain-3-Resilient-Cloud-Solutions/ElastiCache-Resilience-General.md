# ElastiCache Resilience - DOP-C02 Exam Notes

**What it is:** In-memory data store (Redis/Valkey, Memcached). The resilience story is two features people constantly confuse.

**Exam-testable facts:**
- **Redis replication group:** 1 primary + up to 5 read replicas. With **automatic failover**, the replica with the least replication lag is promoted to primary on failure; the primary/reader DNS endpoints follow automatically.
- **Multi-AZ, failover, and replicas move together:** Multi-AZ needs automatic failover, which needs a replica.
- **Multi-AZ ≠ cross-Region.** Multi-AZ survives node/AZ failure *within one Region*. Cross-Region DR is **Global Datastore**: async replication (typically sub-second lag), secondaries serve local reads — but regional failover is **manual** (promote the secondary yourself).
- **Cluster mode disabled** (single shard): full Redis command support, vertical scaling only. **Cluster mode enabled** (up to 500 shards): horizontal write scaling; multi-key commands need all keys in one hash slot (use `{hash tags}`).
- **Exam trap:** "automatic cross-Region failover for ElastiCache" does not exist.

**Sources:** https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/redis-ug.pdf

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
