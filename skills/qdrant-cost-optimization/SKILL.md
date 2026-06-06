---
name: qdrant-cost-optimization
description: "Reduces Qdrant infrastructure costs and resource usage. Use when someone asks 'how to reduce costs', 'Qdrant is too expensive', 'high memory bill', 'how to reduce replicas', 'over-provisioning shards', 'bulk upload is slow and expensive', or 'how to cut infrastructure costs'. Also use when optimizing for resource efficiency rather than raw speed."
---

# Reducing Qdrant Infrastructure Costs

Most Qdrant cost overruns come from two sources: replication factor set higher than necessary and resharding triggered by under-provisioned shards at creation. Check these first before tuning memory or indexing.

## Write Costs Are Too High

Use when: CPU or disk costs are scaling linearly with data volume and write rate.

Every update goes to **all replicas unconditionally**. Replicas provide read availability and fault tolerance, not write throughput. 5 replicas = 5x more CPU and disk I/O for every write.

- Set `replication_factor` to the minimum needed for your availability SLA
- `replication_factor: 1` is viable for non-production or read-only pipelines
- `replication_factor: 2` is the practical production minimum (can lose 1 node without data loss)

See [Replication](https://skills.qdrant.tech/md/documentation/distributed_deployment/?s=replication)


## Resharding Is Too Expensive

Use when: shard count needs to change after initial deployment, or planning a new collection.

Resharding moves a proportional fraction of data between nodes, runs dual-write during transfer, and can take days or weeks on large collections. Time estimate: 3→4 shards ≈ 25% of original upload time.

**Prevention: over-provision shards at collection creation:** Formula: `shard_count = max_expected_nodes × shards_per_node`. Example: planning to scale to 12 nodes with 2 shards/node → create 24 shards up front. Extra empty shards are cheap; resharding is not.

See [Sharding](https://skills.qdrant.tech/md/documentation/distributed_deployment/?s=sharding)


## Memory Bill Is Too High

Use when: RAM costs dominate the infrastructure bill, or memory usage grows faster than data volume.

Keep HNSW in RAM (random-access graph traversal requires it). Text, keyword, and B-tree indexes can be offloaded to disk. Raw vectors can be compressed with quantization:

| Method | Compression | Best for |
|--------|-------------|----------|
| Scalar (float32 → int8) | 4× | General use; minimal accuracy loss |
| Binary (1–2 bits/component) | Up to 32× | High-dimensional, centered distributions |
| TurboQuant | Up to 32× | Strong recall across most embedding models |
| Product Quantization | Up to 64× | Extreme memory minimization |

`rescore: true` re-evaluates top candidates against original vectors. In low-cardinality/filtered workloads, test `rescore: false`: the float32 re-evaluation may be unnecessary.

**Matryoshka dimensions:** if your model supports it (e.g., `text-embedding-3-large`), truncate dimensions: half the dimensions ≈ half the vector memory.

See [Quantization](https://skills.qdrant.tech/md/documentation/manage-data/quantization/)

See [Memory usage optimization](../qdrant-performance-optimization/memory-usage-optimization/SKILL.md)


## Indexing Costs Are Unexpectedly High

Use when: HNSW rebuild time or resource usage grows faster than data volume.

Not all payload indexes cost the same. Text indexes with large vocabularies generate thousands of HNSW sub-graphs per segment rebuild. UUID and high-cardinality fields are similarly expensive.

- Only create `FullTextIndexParams` on fields you actually filter by
- Prefer keyword or integer indexes over text indexes where possible
- Segments under 10 MB are always full-scanned: no index is built for them

See [Payload indexing](https://skills.qdrant.tech/md/documentation/manage-data/indexing/?s=payload-index)


## What NOT to Do

| Anti-Pattern | Problem | Fix |
|--------------|---------|-----|
| `wait=true` on every bulk operation | 10–100× write cost | `wait=false` for all, `wait=true` only at end |
| High `replication_factor` for write-heavy workloads | Linear write cost increase | Minimize replicas |
| No quantization on large datasets | 4× unnecessary memory | Enable quantization |
| Frequent resharding | Days/weeks of dual-write overhead | Over-provision shards at creation |
| Text indexes on high-cardinality fields not used for filtering | Exponential HNSW rebuild cost | Only index fields you filter on |
| Heterogeneous cluster machines | Slowest node caps throughput | Use uniform instance types |
| `rescore: true` in low-cardinality workloads | Unnecessary float32 re-evaluation | Test `rescore: false` with recall validation |
