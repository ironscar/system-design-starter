# Data Latencies

- This document outlines the rough latencies of different kinds of data operations across different systems
- This can be used to make informed design decisions with rough numbers

## Estimated Network Latency

| Operation     | RTT Latency (ms)   |
|---------------|--------------------|
| Same AZ       |  0.25              | 
| Cross AZ      |  2                 |
| Cross Region  |  200               |
|

## Estimated Write Latency

|  Operation             |  Single Write ACK Latency  |       Fsync ACK Time        |    Network RTT       |
|------------------------|----------------------------|-----------------------------|----------------------|
|  In-memory             |   100ns                    |            N/A              |      No              |
|  Disk HDD              |   2ms                      |     (ACK before fsync)      |      No              |   
|  Disk NVMe SSD         |   20us                     |     (ACK before fsync)      |      No              |
|  Disk SATA SSD         |   20us                     |     (ACK before fsync)      |      No              |
|  Redis default         |   100ns                    |     (ACK before fsync)      |      Yes             |
|  Redis AOF always      |   300us                    |     (ACK after fsync)       |      Yes             |
|  Redis RDB             |   100ns                    |     (ACK before fsync)      |      Yes             |
|  Postgres default      |   2ms                      |     (ACK after fsync)       |      Yes             |
|  Postgres sync = off   |   30us                     |     (ACK before fsync)      |      Yes             |
|  ScyllaDB default      |   150us                    |     (ACK before fsync)      |      Yes             |
|  ScyllaDB batch        |   4ms                      |     (ACK after fsync)       |      Yes             |
|

### Interesting Caveats

- NVMe SSD provides faster ACK than Redis default (AOF everysec)
  - the actual Redis write ACK is faster but loses out when network latency is taken into account
  - NVMe SSD also provides better durability guarantees as fsync to disk happens in async immediately after ACK
- NVMe SSD and SATA SSD seem similar in terms of latency numbers
  - but NVMe SSDs have a bigger queue depth allowing more concurrent writes
  - more concurrent writes implies better throughput if using that concurrency
- Redis AOF always is almost never used in production as it does fsync synchronously for per-write durability
  - this increases the latency a lot and we could just use a dedicated database for this
- Redis RDB saves entire data to disk after a much bigger duration (default 1 hour)
  - it is worse for durability and for each save, Redis sees a spike
  - it is faster to load from RDB than AOF though
  - the actual snapshot operation also takes time as it is not just an fsync operation
- Postgres write latency includes multiple things like locking, coordination etc in addition to fsync
  - this results in higher latency but offers guaranteed durability
  - its also faster than ScyllaDB when compared with the same ACK setup
  - ScyllaDB default however ACKs before fsync unlike Postgres
- Latency relationship with data size is as follows
  - small writes upto 4kb have negligible effect on latency
  - SSDs start handling multi-page writes for data greater than 4kb
  - Redis/Postgres/Scylla start having an effect at 1MB or greater due to serialization and WAL overhead (where applicable)
  - these latencies also don't take into account index updates that happen for databases like Postgres

## Estimated Read Latency [TODO]
