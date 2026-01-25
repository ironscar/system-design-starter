# URL Shortener

## Problem Description

- Design a system that generates unique short URLs for given long URLs, ensuring efficient storage and quick retrieval. 
- Focus on the encoding algorithm, database design, and how to handle high read-write ratios and potential collisions.

## Functional Requirements

- Given a long URL, generate a short URL for it
- The short URL generated must always go to this long URL only => minimize collisions

## Non-functional Requirements

- The one-time process of generating the initial short URL can take some time
- The process of getting the long URL back from the short URL must be fast
- The short URLs should be as short as possible
- Make sure that the same long URL does not generate a new short URL and gets duplicated in database
- We may require a way to decommission URLs whose corresponding short URLs maybe reused for new long URLs
- System needs to be highly available and low latency regardless of geographical location

## Scale Considerations

- There are about 3.5 million domains and then each domain will further have more path extensions with fewer encodings

---

## Solution Alternatives

### Solution 1

- Given a long URL like `www.ti.com/product/OPA333/part-details/OPA333AIDCKR` (51 chars)

#### Assumptions

- Not handling query or hash parameters (these will just be forwarded during redirect but not saved during creation of short URL)
- First letter after first `.` in URL is letter (but it would be easy enough to extend to digits and case-sensitive etc)
- Write frequency is low enough for a single async queue and processor to handle it (could attempt to let them know how many jobs in queue or send them result later somehow)
- Assuming number of chunks is 10 (never goes above it) as we partition based on it which cannot change at runtime
  
#### Thoughts

- break the URL into chunks based on `/`
- chunks generated are as follows: `[www.ti.com, product, OPA333, part-details, OPA333AIDCKR]`
  - thus a domain becomes the first chunk
    - assuming we use `[0-9a-zA-Z]` encoding (there will be 51 characters available)
    - 51^5 lands us at 0.35M whereas 51^6 lands us at 17B => `6 characters required for domain`
      - sum(51^n, n: 0-5) = 0.35M approx => (allowing any length encodings vs fixed length encodings doesn't change the required number of characters)
      - `ADR1_OPT1`: Have fixed length chunks for domain
      - `ADR1_OPT2`: Have variable length chunks for domain
    - either way searching for domain after the fact would require O(log2(3.5M)) in an indexed flat table
      - `OPTIMIZATION-1`: most websites that start with `www.` have a fairly distinct character after that
        - thus we take the first character after the first `.` and keep it in our short URL as well to help partition the search space
        - so this character becomes the first character of the domain chunk followed by 5 characters
          - this gives us an upper limit of 26*0.35M encodings which is greater than 3.5M
          - this also partitions the original 3.5M flat table into smaller partitions with a lower limit of 3.5M/26 = 130K and average limit of 3.5M/13 = 260K
        - `ADR1_OPT3`: Based on `OPTIMIZATION-1` so first character comes from URL, used as partition key and rest of the 5 digits searched for in that partition using index
          - can be extended on top of either `ADR1_OPT1` or `ADR1_OPT2`
  - every chunk after domain can have variable length
- so final URL becomes something like `www.my.ly/t0y-cx-31-A-13` (= 25 characters)

#### Database design

- `ADR2_OPT1`: Chunks are reused across chunk index as well as domain
  - DB table 1 stores all the encodings for domain with the mappings
    - partitioned using the first character after first `.`
    - indexed on the character encoding
    - also store the latest encoding used for each chunk in distinct columns
  - DB table 2 stores all the encodings, the chunk index of the URL, the domain encoding and their actual chunk mappings for non-domain chunks
    - partitioned using the chunk index of the URL
    - indexed on (domain encoding, chunk index) in this order
  - DB table 3 and 4 store the last fetch date by chunk index and character encoding
    - this will be populated async after each request to evaluate freshness later and maybe free up open records
    - table 4 will also need to store the domain encoding to find unique records
  - This will have smaller search space per URL request but increase overall table size
- `ADR2_OPT2`: Chunks are reused only across chunk index
  - DB table 1 stores the latest encoding to be used for each chunk index (0 for domain and so on)
  - DB table 2 stores the partition key, 5 character encodings for domain with the actual mappings
    - partitioned using the first character after first `.`
    - indexed on the character encoding
    - indexed on the mapping
  - DB table 3 stores the chunk index as partition key, all character encodings and actual mapped value of chunk
    - partitioned on the chunk index 
    - indexed on the character encoding
    - indexed on the mapping
  - DB table 4 and 5 store the last fetch date by chunk index and character encoding
    - these will be partitioned by first character or chunk index respectively
    - this will be populated async after each request to evaluate freshness later and maybe free up open records
  - DB table 6 and 7 stores a list of reusable encoding from defunct URLs and a status (REUSED or UNUSED)
    - these will be partitioned by first character or chunk index respectively (for now we wont partition on status)
    - this will be populated by the background URL cleanup job
  - This will have larger search space per URL request but decrease overall table size

#### Insert new URL to create short URL

- First, split the url into the chunks by `/` and associate each chunk to the chunk index
  - we get partition keys as `domain = www.ti.com -> t`, `product -> 1`, `OPA333 -> 2`, `part-details -> 3`, `OPA333AIDCKR -> 4`
- Then we need to check if each of these chunk mappings already exist, so lets parallelize that with threads
  - no need to wait for threads to complete as we will send server-sent-event (SSEs) later
  - depending on write frequency, spawning fixed threads per request may starve system resources so instead we use a pool that may increase latency but keeps system operational
  - depending on `ADR2`, we may have to process just domain first `OPT1` or can do all in parallel `OPT2`
  - each thread does the following:
    - search in corresponding chunk tables with their partition keys and actual URL chunk value to see if there is existing record already?
    - if existing record => get the associated encoding
    - if not existing record => forward new insert to queue with (chunk value, partition key for chunk) args 
      - need to include domain encoding and corresponding latest encoding if `ADR2_OPT1`
      - if `ADR2_OPT1` => send all insert requests to the queue as a single item so that the main processor can break it down and combine it back due to domain dependency
  - we can get some chunks with mapped values or no chunks with mapped values so return the corresponding URL back
    - if `www.ti.com` and `part-details` have mappings, it will return `{0: t0y, 3: A}` (map of chunk index to encoding)
- From here, we will follow a CQRS pattern
  - since first-time writes can afford to take some time, we can afford to let this be async
- Async job 1 is the main process which keeps ingesting from queue and does the following:
  - `ADR2_OPT1` => it recieves the list of all chunks to be processed for a request at a time
    - each chunk can be processed as a thread here that we know needs to be a new insert
      - `ADR2_OPT1` just needs domain to be processed first and then rest of them can be parallelized as their encoding sequences are independent
  - `ADR2_OPT2` can recieve each chunk as an independent item and process it
    - there could be distinct processors for each chunk index so that it could be parallelized
  - insert into tables the latest encoding 
    - table 2 or 3 if `ADR2_OPT2` and table 1 or 2 if `ADR2_OPT1`
    - (use current latest encoding from args if `ADR2_OPT1`) with current chunk value using the partition key
    - (select from table 1 using chunk index if `ADR2_OPT2`) with current chunk value using the partition key
  - trigger an SSE to client with chunk index and encoding like `{2: 31}`
    - at this point client has the new chunk encoding and every time it receives one, it checks if there are any pending chunks
    - if no pending chunks, it provides the user with the final short URL
  - update the computed next encodings using the current latest encoding (could create a DB function for it) after the SSE
    - the domain record in table 1 if `ADR2_OPT1`
    - table 1 if `ADR2_OPT2`
- Total time complexity:
  - split = `O(N)` where `N` is number of chunks
  - search for existing mapping depends on `ADR2` (taking into account parallelization)
    - if `ADR2_OPT1` => `O(log2(Ci) + 2log2(D/26))` per thread where `Ci` is number of records per chunk and domain encoding
    - if `ADR2_OPT2` => `O(log2(Cj) + log2(D/26))` per thread where `Cj` is number of records per chunk
    - worst case is all of them are non-existing but we end up searching entire space
  - insert for new mapping per chunk may depend on `ADR2` (taking into account parallelization)
    - if `ADR2_OPT1` => `O(2log2(D/26) + log2(Ci*D/26))` per request
      - insert new domain = `O(log2(D/26))`
      - insert new chunk after domain in parallel = `O(log2(Ci*D/26))` per chunk
      - update latest encoding at domain record all at once = `O(log2(D/26))`
    - if `ADR2_OPT2` => `O(log2(Ci*D/26) + N)` per request
      - insert new domain in parallel = `O(log2(D/26))`
      - insert new chunk in parallel = `O(log2(Ci*D/26))` per chunk
      - update latest encoding at table 1 for each chunk = `O(N)`
    - assume Ci = 10 and N = 5
      - `ADR2_OPT1` complexity = `O(54)`
      - `ADR2_OPT2` complexity = `O(30)`
  - concatenation complexity = `O(N)`
  - final insert complexity
    - `ADR2_OPT1` = `O(64)`
    - `ADR2_OPT2` = `O(40)`

#### Fetch actual URL from short URL

- Comes to the page of the short URL website and then sends a REST request to the short URL backend
- First, split the url into the chunks by `-` and associate each chunk to the chunk index
  - we get the following chunks and partition keys: `0yB03 -> t, cx -> 1, 31 -> 2 , A -> 3, 13 -> 4`
- Then for each chunk, submit the jobs to a thread pool with (chunk encoding, partition key for chunk) args
  - no need to wait for threads to complete as we will send SSEs later
  - `ADR2_OPT1` may require domain chunk to be processed first before parallelizing
  - each thread does the following:
    - select from table 2 or 3 based on partition key and indexed chunk encoding value to get the unique chunk value
    - trigger an SSE with the chunk encoding and chunk index like `{2: 31}`
      - at this point client has the new chunk encoding and every time it receives one, it checks if there are any pending chunks
      - if no pending chunks, it concatenates the chunks by order and redirects to the actual URL
      - if any chunk index returns with null implies that short URL mapping doesn't exist and we send user to `Page does not exist` screen
    - insert into table 4 or 5 based on partition key, the current timestamp and timezone of the request after the SSE
- Total time complexity:
  - split = `O(N)` where `N` is number of chunks
  - search depends on `ADR2` (taking into account parallelization)
    - if `OPT1` =>
      - domain => `O(log2(D/26))` where D is number of domains and 26 first letters form the partition key
      - other chunks => `O(log2(D/26) + log2(Ci))` where `Ci` is the number of records per chunk index and domain encoding
      - total = `O(2log2(D/26) + log2(Ci))` as domain has to happen first and the others can go in parallel after that
    - if `OPT2` =>
      - domain => `O(D/26)` where D is number of domains and 26 first letters form the partition key
      - other chunks => `O(log2(D/26) + log2(Cj))` where `Cj` is the number of records per chunk index and domain encoding
      - total = `O(log2(D/26) + log2(Cj))` as domain can happen in parallel
    - `Cj >> Ci` => `log2(Cj) > log2(Ci)`
      - assume average chunk per domain encoding = 10 => `Ci = 10` and `Cj = Ci*(D/26) = 1.3M`
      - choose `ADR2_OPT2` if `log2(Ci*D/26) - log2(Ci) < log2(D/26)` => `log2((Ci*D/26)/Ci) < log2(D/26)` => `log2(D/26) < log2(D/26)` => both are more or less equal
  - concatenate = `O(N)`
  - final search complexity = `O(2N + log2(D/26) + log2(Cj))` (assuming Ci = 10 and N = 5 => `O(47.4)`)

#### Cleanup unused mappings

- A monthly background job can look at all the last fetch dates in table 4 or 5 and find the ones which are more than 1 year old (could be configurable both in schedule and retention)
- If a particular chunk is old, insert the encoding into Table 6 or 7 based on partition key with status as UNUSED
- Delete all items that are marked with status REUSED (after they have been replicated for reporting somewhere perhaps)
- Changes before SSE during insert flow:
  - could do an A/B testing where a random number from 1 to 10 (if smaller than 3 (configurable maybe)) can decide whether to use sequence or use reusable encodings
  - depending on that, it can query Table 6 or 7 accordingly and pick up an encoding where status = UNUSED (just select 1, order unimportant)
  - update this encoding record on Table 2 oe 3 accordingly (its guaranteed to exist so we can just overwrite it)
  - update this encoding record on Table 6 or 7 accordingly to status REUSED
- The point of having the `status` column is that we can have a reporting job that can check how many encodings actually got reused etc

#### ADR Selection

- SELECTED `ADR1_OPT3 (with underlying OPT1)` for similar encoding, shorter URLs and wider encoding space available
  - makes URLs grow only if the service is being used more
- SELECTED `ADR2_OPT2` for comparable read complexity and better insert complexity
  - while `ADR2_OPT1` had better storage efficiency by repeating chunks across domain, we dont know how often that would come into play and its also little more complex
  - `ADR_OPT2` also enables us to have a distinct processor for each chunk index in the CQRS setup so that each chunk can be independent item in queue enabling better packing

#### System limits

- Currently chunk-specific queue consumers handle synchronous insertion
- Path variables could be particularly challenging to deal with by increasing the number of combinations
- More than 10 URL chunks are not supported by system

#### Questions [TODO]

1. How to do caching?
2. How to achieve high availability?
3. How to achieve geo replication?
4. What is the write frequency supported considering single queue and processor with synchronous insertion?
5. How to deal with path variables?
6. Do we use a relational DB or disk-based key-value DB?
7. Should we use bloom filters to check existing?

-------------------

#### TODOs

Make diagrams for:
- Database diagram
- Architecture diagram
- Fetch flow sequence diagram
- Insert flow sequence diagram

---
