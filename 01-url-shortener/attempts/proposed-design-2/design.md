# Post Peer Review Design

- Refer https://miro.com/app/board/uXjVGFBokdc=/
- We can try to document the entire miro board as `.md` files so as to be exportable [TRY]

## Database selection findings

### ScyllaDB

- ScyllaDB can do INSERT IF NOT EXISTS but it attempts to check across all shards for presence (inherently idempotent) but takes a hit for throughput compared to simple writes
  - INSERT IF NOT EXISTS returns an `applied: true` if row is inserted and `applied: false` along with existing row if row exists already
  - if for bulk and if even of the rows exist, the entire batch fails
  - it doesn't return rows while doing insert in same query either, it has to be a separate service-level query to fetch the row from DB
- ScyllaDB doesn't have the concept of sequences and even though it does have a counter type, we cannot increment and return the counter in the same statement
- ScyllaDB doesn't have autoincrement and as a result we have to use functions like `now()` to generate time-ordered UUIDs ensuring uniqueness or generate it on the client
- ScyllaDB tends to outperform traditional relational databases in blind INSERTS and non-JOIN SELECTS at high throughputs, and scales linearly by core
- ScyllaDB doesn't support inner queries either, like what we might need during the sequence recycling setup, so these have to be handled separately at service-level
- ScyllaDB internally uses Murmur hash to decide its partition key to tell what data lies in what shard
  - searching via this is fast but searching without it is disallowed
  - we can fetch the murmur hash value of the row using the `token()` function in query
  - we can however create materialized views with a different partition key which are automatically updated when the main table updates

#### Write Process

- Generate UUID on service
- Increment current counter value
- Get current counter value (how to get counter per shard? [CHECK])
- Blind-insert row with location key + UUID as partition key, current counter value and actual URL
- Use scylla driver libraries to get the murmur hash value that was inserted and encode it to get the short URL fragment
- Prefix with location key and suffix with counter to get final short URL and return
