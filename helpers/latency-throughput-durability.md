# Latency vs Throughput vs Durability

- This document compares the latency, throughput and durability of data for each type of write path that can be used
- We can store data either
  - `on node` (sync to disk after ACK)
  - `on shared cache` (sync to disk every second + RTT)
  - `on database` (sync to disk before ACK + RTT)
  - `on queue` (sync to disk after ACK + RTT)
- Disk corruption is far more rare than server failures that only lose RAM data
- If overall write througput still needs to increase regardless of the data storage mechanism here, refer to `helpers/high-write-throughput-strategies.md`

## Database

- Start with storing in database because it has best durability as it sends ACK only after saving to disk
  - this will also avoid distributed coordination complexities as database is shared and services are stateless
  - any service can read the data
  - take into account additional replications (if any) from the primary DB has RTT cost and ACK waits
- As per `helpers/data-latency-reference.md`, we will assume RTT within AZ as 250us and write latency as 2ms
  - so a single write to a single database will take 2.25ms
- Here latency is also highest and therefore throughput would be lesser
- We don't lose any data if the server fails as the data is in disk
  - if disk is also corrupted, then we have to start considering additional replications which will also add up the latency
- Examples: Postgres

## Shared Cache

- Consider storing in cache next as its less durable but still shared, so distributed coordination complexity is avoided to some degree
  - any service can read the data
  - take into account additional replications (if any) from one instance has RTT cost
- As per `helpers/data-latency-reference.md`, we will assume RTT within AZ as 250us and write latency as 20us
  - so a single write to a single instance will take 0.252ms
- Here latency is lower and therefore throughput would be higher
- We may lose data upto a second if the server fails as data is synced to disk every second
  - if we want to get better guarantees on that 1 second loss, then we have to start considering additional replications which will also add up the latency
  - if disk is also corrupted, then we have to start considering additional replications which will also add up the latency
- Examples: Redis (AOF everysec)

## Per Node

- If we need even more durability at the same throughput or even higher throughput, then we consider storing on node
  - this will have to flush data to a shared database at some interval to avoid losing data on disk corruption
  - a service that needs to read this data needs to know which node the data is on which adds to the distributed coordination complexity (refer to `herlpers/distributed-coordination.md`)
    - if flush to disk happens at some interval, we can check older data in shared database and latest data in specific instances
- As per `helpers/data-latency-reference.md`, we will assume write latency as 20us and RTT = 0
  - so a single write to a single instance will take 0.002ms
- Here latency is lowest and therefore throughput is highest
- We may lose data upto 0.25ms as SSDs take that much time to sync to disk after sending ACK
  - if we want to get better guarantees on that 0.25ms loss, then we have to start considering additional replications which will also add up the latency and bring back RTT costs
  - if disk is also corrupted, then we have to start considering additional replications which will also add up the latency

## Queue

- This works similar to Per-node but is shared across all instances
  - avoids distributed coordination complexity but brings back network RTT
- As per `helpers/data-latency-reference.md`, we will assume write latency as 400us and RTT = 2*250us
  - so a single write to a single instance will take 0.95ms
- Latency is faster than database but slower than shared cache, and throughput follows similar semantics
- Losing data is unpredictable as Kafka fsync depends on OS flush
  - if we want to get better guarantees, then we have to start considering additional replications which will add another round of network RTT cost
  - if disk is also corrupted, then we have to start considering additional replications which will add another round of network RTT cost
- Examples: Kafka (acks=all)
