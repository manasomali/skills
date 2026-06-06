---
name: qdrant-write-path-optimization
description: "Diagnoses and optimizes Qdrant write path performance. Use when someone asks 'how to speed up bulk uploads', 'what does the wait parameter do', 'write consistency explained', 'WAL queue full', 'updates not appearing in search', 'how many shards for write throughput', or 'replica write cost'. Also use when write latency is high or throughput is insufficient."
---

# What to Do When Qdrant Write Performance Is a Problem

Every write flows through: routing → WAL → update worker → segments. Understanding this path is key to diagnosing write issues.

## Bulk Uploads Are Slow

Use when: write throughput is lower than expected during bulk ingestion.

The `wait` parameter is the single most impactful lever. Controls when the operation returns to the caller.

| Value | Returns after | Latency | Use when |
|-------|--------------|---------|----------|
| `wait=false` | Written to WAL | ~1–10ms | Bulk uploads, high-throughput ingestion |
| `wait=true` | Update worker applied it to segments | ~100ms | Need read-your-writes guarantee |

**Default differences:** REST API defaults to `wait=false`. Python client defaults to `wait=true` (changed because users complained updates weren't appearing in searches immediately after upsert).

**Bulk upload pattern:** send all operations with `wait=false`, send only the final operation with `wait=true`. The last `wait=true` confirms all previous ops were persisted at the cost of one round-trip wait.

See [Points API](https://skills.qdrant.tech/md/documentation/manage-data/points/?s=upload-points) for upsert options.


## Write Throughput Is Not Scaling

Use when: adding more resources or nodes doesn't improve write throughput.

Each shard has exactly one update worker (single-threaded by design, maintains ordering). Shards operate fully independently in parallel.

- **More shards → more write parallelism.** 12 shards on 3 nodes = 4× more write workers than 3 shards on 3 nodes
- **More replicas → more write cost.** Every replica unconditionally processes every update. 5 replicas = 5× more CPU/disk for writes

For write-heavy workloads: increase `shard_number`, minimize `replication_factor`.


## Replica Divergence Is Unacceptable

Use when: replica consistency is a hard requirement (financial, audit, or compliance use cases).

| `write_consistency` | Behavior | Use when |
|--------------------|----------|----------|
| `1` (default) | Returns OK if ≥1 replica responds | Typical workloads |
| `2+` | Returns error if fewer than N replicas confirm | Replica divergence is unacceptable |

With `write_consistency=1`: if a replica doesn't respond, the op succeeds and that replica is marked dead until recovered. With `write_consistency=2+`: any non-responding replica causes the op to fail: caller must retry.

See [Write consistency](https://skills.qdrant.tech/md/documentation/distributed_deployment/?s=write-consistency)


## Updates Not Appearing in Search Results

Use when: recent writes are not visible in search, or `wait=true` operations are timing out.

The WAL serializes and persists operations before applying them to segments. With `wait=false`, writes fill the queue in RAM. If updates arrive faster than indexing can process, ops queue up and `wait=true` operations may timeout.

- Data is persisted in WAL but may not yet be visible in search results
- ~20,000 non-indexed points per segment: exceeding this temporarily hides points until optimization catches up
- v1.17+ throttles updates automatically when queue depth is too high


## Write Performance Degrades on Already-Indexed Data

Use when: write throughput drops after initial bulk load, as updates target existing indexed points.

When a point in an immutable indexed segment is updated, the old version is tombstoned and the new copy is written to a mutable segment. This triggers more optimizer cycles over time.

- Avoid frequent payload modifications on large indexed collections if write throughput is critical
- Heavy update workloads cause tombstone accumulation, which triggers the vacuum optimizer and additional CPU cycles


## What NOT to Do

- Do not use `wait=true` for every operation during bulk uploads: it serializes writes and is 10–100× slower
- Do not add replicas expecting write throughput improvement: replicas multiply write cost
- Do not assume updates are immediately visible after `wait=false`: they may still be in the update worker queue
- Do not set `write_consistency=2+` unless replica divergence is genuinely unacceptable: it increases error rate under any node hiccup
