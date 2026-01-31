# Post Analysis

## Final solution feedbacks [TODO]

---

## Peer review feedbacks

### Technical things you could have considered

1. Consider analytics of how many times a specific URL is hit and how would it work when some URLs are cached
2. Consider if we want to protect from botnets which can figure out all previous encodings in a sequential setup and have 100% efficacy in blocks
3. Should we consider rate limiting
4. Consider how much storage is required and how about hot storage and cold storage?
5. Should have thought about the different kind of redirect status codes
6. Consider using BIGINT sequence storage for the encodings
7. Consider keeping next set of unused sequenced encodings as buffer in replicated Redis and then fetch it from a master DB ahead of time when they are about to run out so that writes can scale out
8. Consider encoding the sequence of the URL instead of encoding the URL so that super long URLs don't cause collisions depending solely on the string

### Things to look into

1. What are some strategies for avoiding hash collisions?

### Things you could have done better

1. Parameter encoding is one of the major reasons why URLs are shortened so you cannot just assume thats not part of the problem
2. If two inserts have same partition key one after the other in queue, then the insert service ends up generating two different encodings for same URL segment value, thus violating idempotency
3. The solution explanation ended up being too complicated for anyone to understand which is counterproductive
4. Ended up thinking about chunking the URL to reuse space without doing the math on actual storage and then without thinking of actual sequence limits, ended up encoding sequence with Base 62 leading to possible overengineering

---
