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

## Scale Considerations

- How many URLs is the system designed to handle?

---
