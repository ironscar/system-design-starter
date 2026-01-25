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
- The short URLs should be as short as possible for ease of storage
- Make sure that the same long URL does not generate a new short URL and gets duplicated in database
- We may require a way to decommission URLs whose corresponding short URLs maybe reused for new long URLs
- System needs to be highly available and low latency regardless of geographical location

## Scale Considerations

- There are about 3.5 million domains and then each domain will further have more path extensions with fewer encodings

---

## Solution Alternatives

### Solution 1

- Given a long URL like `www.ti.com/product/OPA333/part-details/OPA333AIDCKR` (51 chars)

## Assumptions

- Not handling query or hash parameters
- First letter after first `.` in URL is letter (but it would be easy enough to extend to digits and case-sensitive etc)
- Write frequency is low enough for a single async queue and processor to handle it
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
        - `ADR1_OPT3`: Based on `OPTIMIZATION-1` so first character comes from URL, used as partition key and rest of the 5 digits searched for in that partition using index [RECOMMENDED]
          - can be extended on top of either `ADR1_OPT1` or `ADR1_OPT2`
  - every chunk after domain can have variable length
- so final URL becomes something like `www.my.ly/t0yB03-cx-31-A-13` (= 27 characters)

#### Database design

- `ADR2_OPT1`: Chunks are reused across chunk index as well as domain
  - DB table 1 stores all the encodings for domain with the mappings
    - partitioned using the first character after first `.`
    - indexed on the character encoding
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
    - this will be populated async after each request to evaluate freshness later and maybe free up open records
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
      - need to include domain encoding if `ADR2_OPT1`
  - we can get some chunks with mapped values or no chunks with mapped values so return the corresponding URL back
    - if `www.ti.com` and `part-details` have mappings, it will return `{0: t0yB03, 3: A}` (map of chunk index to encoding)
- From here, we will follow a CQRS pattern
  - since first-time writes can afford to take some time, we can afford to let this be async
- Async job 1 is the main process which keeps ingesting from queue and does the following:
  - each chunk is processed as a thread here that we know needs to be a new insert
  - insert into table 2 or 3 the latest encoding (select from table 1 using chunk index if `OPT2`) with current chunk value using the partition key
  - trigger an SSE to client with chunk index and encoding like `{2: 31}`
    - at this point client has the new chunk encoding and every time it receives one, it checks if there are any pending chunks
    - if no pending chunks, it provides the user with the final short URL
  - insert into table 1 the computed next encoding using the current latest encoding (could create a DB function for it) after the SSE
- Total time complexity:
  - split = `O(N)` where `N` is number of chunks
  - search for existing mapping depends on `ADR2` (taking into account parallelization)
    - if `OPT1` => `O(log2(Ci) + 2log2(D/26))` per thread where `Ci` is number of records per chunk and domain encoding
    - if `OPT2` => `O(log2(Cj) + log2(D/26))` per thread where `Cj` is number of records per chunk
    - worst case is all of them are non-existing but we end up searching entire space
  - insert for new mapping per chunk may depend on `ADR2` (taking into account serial processing)
    - if `OPT1` => `O(log2(D/26) + (N-1)*log2(N*D/26))` per thread
      - complexity of figuring out the next combination needs to be included
        - for domain, `O(D/26)` and for each chunk after that, `O(log2(N*D/26) + Ci)`
      - total for `OPT1` => `O(log2(D/26) + N*log2(N*D/26) + Ci + D/26)`
    - if `OPT2` => `O(N*(N + log2(D/26)))` per thread
    - assume Ci = 10 and N = 5
      - `OPT1` complexity = `O(134738)` (majorly loses out due to linear complexity of figuring out next combination)
      - `OPT2` complexity = `O(110)`
  - concatenation complexity = `O(N)`
  - final insert complexity assuming `OPT2` = `O(120)`

#### Get actual URL from short URL

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

- A background job can look at all the last fetch dates in table 4 or 5 and find the ones which are more than 1 year old (could be configurable perhaps)
- Look at how to delete the record here and reassign it in actual tables `Questions: reuse defunct url` [TODO]

#### ADR Selection

- SELECTED `ADR1_OPT3 (with underlying OPT1)` for shorter URLs and wider encoding space available
- SELECTED `ADR2_OPT2` as while its read complexity is similar, the insert complexity is way better due to storing the last used combination

#### System limits

- Currently one queue handles synchronous insertion
- Path variables could be particularly challenging to deal with by increasing the number of combinations
- More than 10 URL chunks are not supported by system

#### Questions [TODO]

1. How to reuse defunct URLs?
2. How to mix `ADR2_OPT1` and `ADR2_OPT2` to get best of both worlds?
3. Should we use bloom filters to check existing?
4. Which options to select and why?
5. How to do caching?
6. How to achieve high availability?
7. How to achieve geo replication?
8. What is the write frequency supported considering single queue and processor with synchronous insertion?
9. How to deal with path variables?
10. Do we use a disk-based key-value DB or relational DB?

-------------------
