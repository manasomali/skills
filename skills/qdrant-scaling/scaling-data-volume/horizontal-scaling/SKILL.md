---
name: qdrant-horizontal-scaling
description: "Diagnoses and guides Qdrant horizontal scaling decisions. Use when someone asks 'vertical or horizontal?', 'how many nodes?', 'how many shards?', 'how to add nodes', 'resharding', 'data doesn't fit', or 'need more capacity'. Also use when data growth outpaces current deployment."
---

# What to Do When Qdrant Needs More Capacity

Vertical first: simpler operations, no network overhead, good up to ~100M vectors per node depending on dimensions and quantization. Horizontal when: data exceeds single node capacity, need fault tolerance, need to isolate tenants, or IOPS-bound (more nodes = more independent IOPS).

## Starting a New Distributed Deployment

Use when: setting up a new cluster from scratch.

- 3 nodes, 3 shards with `replication_factor: 2` is the minimal production baseline for zero-downtime scaling
- Minimum 3 nodes is required for consensus and fault tolerance: with 2 nodes, losing 1 causes downtime for collection operations
- `replication_factor: 2` means each shard has 1 replica, so 2 copies exist: required for zero-downtime maintenance


## Choosing Initial Shard Count

Use when: planning a new collection before any data is loaded.

Shards are the unit of data distribution. More shards allows more nodes and better distribution; fewer reduces overhead but limits horizontal scaling. Recommended range: 6-12 shards for a 3-6 node cluster.

**Over-provision formula:** `shard_count = max_expected_nodes × shards_per_node`. Example: expect to grow to 12 nodes with 2 shards/node → start with 24 shards on 3 nodes. Scaling to 4, 6, or 12 nodes later requires no resharding. Extra empty shards are cheap.


## Resharding After Initial Deployment

Use when: shard count isn't evenly divisible by node count, causing uneven distribution, or need to rebalance.

Resharding is expensive and time-consuming: use as a last resort. Qdrant uses a hash ring so only points that would map to the new shard are transferred. Adding a 4th shard to a 3-shard cluster moves ~25% of data. Time estimate: 3→4 shards ≈ 25% of original upload time.

**During resharding:** queries go to whichever shard has the data; inserts/updates/deletes go to both shards. No downtime, but dual-write overhead exists for the duration.

**Resharding operations (v1.17+):**
- Cluster-wide API shows progress and estimated completion time
- Cancellable at any point with no data loss (cancel, not pause)
- Only 1 resharding operation can run at a time; supports both increasing and decreasing shard count

- Available in Qdrant Cloud [Resharding](https://skills.qdrant.tech/md/documentation/distributed_deployment/?s=resharding)
- Resharding is not available for self-hosted deployments.

Better alternatives: over-provision shards initially, or spin up a new cluster with correct config and migrate data.


## Indexing Can't Keep Up With Write Rate

Use when: indexing falls behind a high constant update rate and search quality degrades.

- **5 read + 2 write replica** pattern: write replicas absorb all updates and build indexes; read replicas serve search queries
- Minimum 2 write replicas for resilience
- Only worthwhile under extreme write pressure: adds significant resource overhead

This is different from standard replication: it explicitly routes writes and reads to different replica subsets.


## What NOT to Do

- Do not jump to horizontal before exhausting vertical (adds complexity for no gain)
- Do not set `shard_number` that isn't a multiple of node count (uneven distribution)
- Do not use `replication_factor: 1` in production if you need fault tolerance
- Do not add nodes without rebalancing shards (use shard move API to redistribute)
- Do not scale down RAM without load testing (cache eviction causes days-long latency incidents)
- Do not hit the collection limit by using one collection per tenant (use payload partitioning)
