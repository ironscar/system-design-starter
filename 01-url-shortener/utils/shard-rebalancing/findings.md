# FINDINGS

## COMPLEXITY

### WRITE COMPLEXITY

- MurmurHash of each URL
- Then comparing the URL hash with the hashes of all virtual shards and figuring out the physical shard and put it in the corresponding queue
- Then actual complexity of batch insert

### READ COMPLEXITY

- Decode physical shard index from URL
- Then actual search complexity of the rest of the decoded URL

----------------------------------------------------------------------------------------------------------------------------------

## SPIKE HANDLES

- 100 shards can be added/removed with roughly 5% migrations
- 100 shards handle 2K req/s per shard = 200K req/s total
- 5% of data = 275GB data
- data migration can be handled in parallel for each source which at max = 0.07% = 3.85GB
- rebalancing process would be like (total time ~3 hours)
    - spin up new shard instances if required (30 minutes)
    - spin up compute in parallel for hash ring updates and data migration (30 minutes) based on number of shards
    - send new hash ring updates to all service instances (60 minutes)
    - migrate at max 3.85GB records from each shard in separate compute cluster (60 minutes)
    - tear down compute cluster and any old shard instances if required
- So it handles spike rates of 200K RPS change in 3 hours
    - services can start writing to new shards within 2 hours and requests may end up with double latency in the final hour

----------------------------------------------------------------------------------------------------------------------------------

## ASSUMPTIONS

- 1000000 keys
- 500 physical shards -> 400 physical shards
- 10 virtual shards per physical shard on hash ring
- precision = 5

## RESULTS

- Removing collided virtual shards from the old shards leads to 4934 virtual shards but luckily no physical shard is completely removed
    - additionally there were also no empty physical shards after distributing keys
    - so no wasted shards in old state
    - but non-zero chance for some physical shards to get completely removed due to collisions or uneven distribution
- Removing collided virtual shards from the old shards leads to 3957 virtual shards but luckily no physical shard is completely removed
    - additionally there were also no empty physical shards after distributing keys
    - so no wasted shards in new state
    - but non-zero chance for some physical shards to get completely removed due to collisions or uneven distribution
- Number of key migrations required between physical shards = 20.8%
    - key migrations in the basic module setup = 80% so we are nearly four times better
    - maximum migrations from any one physical source = 0.4%
    - 20% is still a lot though removing 1000 shards at once is also not very realistic probably

----------------------------------------------------------------------------------------------------------------------------------

## ASSUMPTIONS

- 1000000 keys
- 5000 physical shards -> 4000 physical shards
- 10 virtual shards per physical shard
- precision = 5

## RESULTS

- No physical shards removed due to collisions
- Physical shards rendered empty due to key distribution in old state = 413 (thats not great utilization)
- Physical shards rendered empty due to key distribution in new state = 151
- Migration percentage = 41.2% and maximum migration from one source = 0.07%

## USING EQUIDISTANT SHARD DISTRIBUTION

- Physical shards rendered empty due to key distribution in old state = 749 (thats worse utilization)
- Physical shards rendered empty due to key distribution in new state = 599 (thats worse utilization)
- Migration percentage = 19.8% and maximum migration from one source = 0.03% (that's better migration statistics)

----------------------------------------------------------------------------------------------------------------------------------

## ASSUMPTIONS

- 1000000 keys
- 5000 physical shards -> 4900 physical shards
- 10 virtual shards per physical shard
- precision = 5

## RESULTS

- Physical shards rendered empty due to key distribution in old state = 413 (thats not great utilization)
- Physical shards rendered empty due to key distribution in new state = 377 (thats not great utilization)
- Migration percentage = 5% and maximum migration from one source = 0.07%

## USING EQUIDISTANT SHARD DISTRIBUTION

- Physical shards rendered empty due to key distribution in old state = 749 (thats worse utilization)
- Physical shards rendered empty due to key distribution in new state = 734 (thats worse utilization)
- Migration percentage = 1.97% and maximum migration from one source = 0.03% (that's better migration statistics)

----------------------------------------------------------------------------------------------------------------------------------

## ASSUMPTIONS

- 1000000 keys
- 4800 physical shards -> 4900 physical shards
- 10 virtual shards per physical shard
- precision = 5

## RESULTS

- Physical shards rendered empty due to key distribution in old state = 339 (thats not great utilization)
- Physical shards rendered empty due to key distribution in new state = 377 (thats not great utilization)
- Migration percentage = 5% and maximum migration from one source = 0.07%

## USING EQUIDISTANT SHARD DISTRIBUTION

- Physical shards rendered empty due to key distribution in old state = 719 (thats worse utilization)
- Physical shards rendered empty due to key distribution in new state = 734 (thats worse utilization)
- Migration percentage = 1.98% and maximum migration from one source = 0.03% (that's better migration statistics)

----------------------------------------------------------------------------------------------------------------------------------

## OPTIMAL SHARD COUNTS

### METRIC DESCRIPTION

- We find the sum of the distance of 95% values that fall below mean and distance of 95% values that fall above mean (more trusty than standard deviation)
- lower the p95 value, closer the number of keys distributed per shard

### 1M KEYS & 1K-6K SHARDS

- For 1M keys (which ought to be handled in 100ms), lets try to find the optimal equidistant-shardCounts and compare it to optimal murmurhash-shardCounts
- For a range of 1000 to 6000, optimal shard count for equi-shard distribution = 5000
    - empty shards = 749
    - p95 deviation = 14 on the left and 72 on the right of mean
- For a range of 1000 to 6000, optimal shard count for murmur-shard distribution = 5700
    - empty shards = 710
    - p95 deviation = 132 on the left and 346 on the right of mean
- Murmurhash did have empty shards at 1800 shards but its p95 value was much greater
    - since the distribution is sparse (1M is not the same as 50B while keeping same shard count), we will ignore the empty shards

### 50M KEYS & 2-20 SHARDS

- This configuration is closer to the distribution statistics if we consider 50B keys (1000x current keys so we can scale shard count accordingly)
- Optimal shard count for equi-shard distribution = 19
    - mean = 2,631,578.95
    - p95 range = 49,627,858
    - p95 shard coverage = 100%
- Optimal shard count for murmur-shard distribution = 19
    - mean = 2,631,578.95
    - p95 range = 2,005,109
    - p95 shard coverage = 100%

----------------------------------------------------------------------------------------------------------------------------------

## CONCLUSIONS

- Equidistant sharding tends to have much better migration statistics but worse key distrbution with the same shard counts
- Considering that ultimately the shards that get the keys are the ones that will handle the write throughput, it makes sense to optimize key distribution
- P95 value is much lesser for Murmur-shard configuration signifying that distribution is more equal across shards than the Equi-shard configuration
    - this was true when the raio of number of keys (5M) to the number of shards (50) were closer to the actual scale (50B keys, 5K shards)
- Murmur-shard configuration also had consistently fewer empty shards than the equi-shard configuration for the same shard counts
- Thus the murmur-shard configuration is better at consistent scale
- In general though, it also looks like p95 is lower for higher shard counts, so how do we find the optimal shard count above which it doesnt make sense
  - look at % of shards that cover P95 range, more the percentage => P95 keys are distributed among more shards thus reducing request load per shard
  - so we maximize the % of shards that cover P95 range and minimize the P95 distribution of keys
  - out of the ones that get 100% coverage, we select the one with the lowest P95 sum value (usually the one with highest shard count but sometimes not)
- We would need to also check if given that distribution, the write throughput of 10M req/s is achievable as read throughput can always be achieved by adding more read replicas
  - 1M keys => 100ms resolution overall where all shards working in parallel
  - assuming 1000 key batch insert and time per insert = 100ms => each shard can do exactly one insert at max efficacy
  - refer to results.log
    - for equi-shards: sweet spot is at 2530 with max = 555 (chosen due to better P95 shard coverage % (95%) and 0 empty shards)
    - for murmur-shards: sweet spot is at 2620 with max = 993 (P95 shard coverage % (90%) and 13 empty shards)
- At extreme write throughput, the equi-shard config seems to be better (as seen from 1M keys example), but this is only about supporting write throughput with about 110 fewer shards
    - eventually, once more keys come in, the distribution for the murmur-shard config will be better (as seen from 50M keys example)
    - the read throughput for the equi-shard config will be much worse (around 20x on the shard with most keys)
    - therefore, the comparison will be offset if the equi-shard config needs just as many shards as read replicas to support the required read throughput
    - assuming 20 threads per replica, 2 replicas per shard and each read is 2ms (50 in 1000ms)
      - equi-shard config read throughput = 2530 * 3 * 20 * 50 = 7.59M reads in 100ms (without taking into account the 20% degradation on hottest shard)
          - adding the 20% penalty as 7.59 * 0.8 = 6.1M reads in 100ms
      - murmur-shard config read throughput = 2620 * 3 * 20 * 50 = 7.86M reads in 100ms
      - the difference = 7.86M - 6.07M = 1.79M
      - number of additional read replicas required for equi-shard config to offset this = 1.79M / (3 * 20 * 50) = 596
      - 110 replicas will help with 0.3M reads per 100ms more => 6.4M reads in 100ms (adding it to base equi-shard config so that shard count for both configs is now equal)
- Overall, if we are good with 64M reads/sec (6.4M reads in 100ms), then we go ahead with equi-shard config, but for higher read throughput, we need to use murmur-shard config
