# Latency vs Throughput vs Durability

- We can store data either
  - `on node` (sync to disk after ACK)
  - `on shared cache` (sync to disk every second + RTT)
  - `on database` (sync to disk before ACK + RTT)
- Disk corruption is far more rare than server failures that only lose RAM data

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
  - a service that needs to read this data needs to know which node the data is on which adds to the distributed coordination complexity
    - if flush to disk happens at some interval, we can check older data in shared database and latest data in specific instances
- As per `helpers/data-latency-reference.md`, we will assume write latency as 20us and RTT = 0
  - so a single write to a single instance will take 0.002ms
- Here latency is lowest and therefore throughput is highest
- We may lose data upto 0.25ms as SSDs take that much time to sync to disk after sending ACK
  - if we want to get better guarantees on that 0.25ms loss, then we have to start considering additional replications which will also add up the latency and bring back RTT costs
  - if disk is also corrupted, then we have to start considering additional replications which will also add up the latency
