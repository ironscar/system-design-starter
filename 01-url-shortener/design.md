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

## Scale Considerations

- How many URLs is the system designed to handle?

---

## Solution Alternatives

### Solution 1

- Given a long URL like `www.ti.com/product/OPA333/part-details/OPA333AIDCKR` (51 chars)
- Steps:
  - break the URL into chunks based on `/`
  - chunks generated are as follows: `[www.ti.com, product, OPA333, part-details, OPA333AIDCKR]`
  - have a sequence in DB where these chunks are inserted one-by-one and the corresponding sequence value is assigned
    - the sequence should start with atleast 2 characters where first character is index of chunk
    - `00` => `www.ti.com`
    - `10` => `product`
    - `20` => `OPA333`
    - `30` => `part-details`
    - `40` => `OPA333AIDCKR`
  - corresponding URL then becomes `my-url-shortener/0010203040`
  - issue is that eventually there could be a chunk mapped to `001` and we will no longer know where to separate
  - so we can assign a specific character to slashes - such as `_` and not use that in the sequence at all
  - final URL becomes `my.ly/00_10_20_30_40` (6 + 14 chars)
  - the chunks table can be partitioned based on index of chunk and then each partition could have its own index
- We can also make the sequence count using base 36 (0-9,a-z) or base 62 (0-9,a-z,A-Z)
  - that way the short URL may look something like `my.ly/0ah_1b2_2c3_3dd_489`
- Given `00_10_20_30_40`, 
  - we split the short URL based on `_`
  - then the first character of each split gives the position of the chunk
    - supports upto 36 or 62 chunks in the URL so that one character can denote it in base 36 or base 62
    - but that means the number of underscores itself is fairly large and so effective chunks supported are lesser
  - the rest of the characters is what we have to find with an index from the partition
- This system is guaranteed to have no collisions as we are doing exact lookups
  - but eventually the length will increase beyond what can be called short
- Concurrency considerations
  - concurrent reads should have no problems as long as the database is appropriately scaled
  - concurrent writes end up being asynchronous inserts so that the sequence can generate safely without collisions serially

#### System limits

- Assume that we dont want more than 15 characters after the initial `/` in the short-url
- If each chunk is atleast 2 characters, most number of chunks supported = 5
  - any 1 chunk may have 3 characters => 2 characters for the actual base 62 encoding
- First character stores position of chunk and second character can store 62 different combinations using base 62
  - if two characters after position character, it can store 62*62 = 3844 combinations
- A URL will have atleast 2 chunks and atmost 5 chunks
  - if 5 chunks => each chunk can have between 2 and 3 characters => 1 to 2 characters for encoding => 62 to 62^2 combinations per chunk
    - total unique combos putting all chunks together = `62^4 * 62^2 = 62^6 = 56.8e9`
- Lets proceed assuming then that total possible combos are 56.8e9 combinations and per chunk cardinality is between 62 and 3844
  - complexity for getting long URL from short URL
    - splitting the short URL into chunks = O(15) assuming max 15 characters
    - then atmost 5 separate threads can hit the DB to get the corresponding value for each chunk
    - each thread will hit a specific partition of the DB based on the position character and use the partition index to find the specific entry
    - read complexity of a B-tree index = O(log2(N)) => O(log2(3844)) = O(11.9) handled in parallel by each thread
    - then each chunk is appended in order and returned as actual URL
    - total time complexity = `O(15 + 11.9 + 5) = O(31.9)`
    - more generally, if N = number of max characters in URL, M = base encoding used and P = number of chunks => `O(N + 2logM + P)`
    - since M is mostly constant here (62), it becomes `O(N + 11.9 + P)`
    - implying linear complexity in terms of the short URL length
  - complexity for generating new short URL from new long URL
    - splitting the long URL into chunks = O(N) assuming URL is N characters long
    - then we can insert the chunks to a DB/queue in a CQRS fashion to make sure no two end up getting the same value = O(P) where P = number of chunks
    - they can then be asynchronously assigned an incremental sequence value converted to base 62 and inserted into actual chunk DB and clear from queue = O(P)
      - the B-tree write complexity is O(log2(N)) => O(log2(3844)) = O(11.9)
      - our complexity will be 5 times this at max = O(59.5)
    - once all done, then a Server-side event can send the corresponding chunk values generated and put it all together = O(P)
    - total time complexity = `O(N + 59.5 + 3P)`
    - this is also mostly linear in terms of the length of the long URL
- Total URLs supported = `56.8 billion` roughly
- Total read complexity = `O(N + 11.9 + P)`
- Total write complexity = `O(N + 59.5 + 3P)` (asynchronous considerations for the first-time generation)

#### Notes

- Initial URLs will be shorter than later URLs
- We are breaking into positional chunks so that we can keep each partition smaller to search in which reduces possibility of reuse

#### Questions [TODO]

1. How to check if current long URL already exists in DB or not?
2. Do you really need the position bit, can we not figure it out from the URL directly?
3. Should we use hash indexing instead since its just 3844 combinations per partition?
4. Is CQRS-based async insert the best way to go?
5. Is there a better way to store the data?
6. Is there a better encoding algorithm?
7. Can we make URLs similar length for old and new entries?
8. How to reuse combinations for defunct URLs?

---
