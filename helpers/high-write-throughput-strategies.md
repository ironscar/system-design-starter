# Read/Write strategies for high write throughputs

- This document discusses how to handle read/write for high write throughputs when just deciding between database, shared cache and per-node write paths (refer `helpers/latency-througput-durability.md`) is not enough
- Regular read/writes may end up being slow for very high throughputs
- In such cases, we can use CQRS to speed up the write path
  - CQRS is command query responsibility segregation
  - this is usable regardless of whether data is being stored in-memory, database or queue 
- If using CQRS, the writer and reader are decoupled and the reader doesn't know when the write operation is actually done
- So if you have to return the result of a processed write while using CQRS, you have following ways to handle it
  - `client-side polling` at some interval to check in desired location if the result is ready or not and return it
  - `event-driven SSE` where the service holds open SSE connections until an event notifies with result and then returns it

## Client-Side Polling Mechanism

- Client sends write request to writer service
- Writer service creates an entry somewhere about this request but doesn't process it
  - this somewhere can be a database, a queue or in-memory
  - this allows the request to not wait for processing which makes the request itself fast and allow more throughput
  - the request doesn't return results however so that must be fetched separately
- If database or in-memory
  - a background async worker polls the database at some defined interval to process all entered requests, preferably in bulk
    - the worker is probably shared across instances if database is shared and per-node if the data is stored in-memory
  - once processing is done, results can be logged back into the database
- If queue
  - the entries are published into queue and consumers process them in a decoupled manner
  - once processing is done, results can be logged back into memory or a database
- The client can then poll at some predefined interval
  - ideally this interval should be such that the first poll happens by when the processing is done so that it only really polls once in general cases
  - if results stored in database
    - the read service queries the database for the corresponding result
  - if results stored in-memory
    - the read service sends an API to the corresponding node that is storing the data in memory
  - then returns results to the client
- No open connections held but some wasted processing during polling if processing is not done by that time

## Event-Driven SSE Mechanism

- Client sends write request to writer service
- Writer service keeps the connection open and then creates an entry somewhere about this request but doesn't process it
  - this somewhere can be a database, a queue or in-memory
  - this allows the request to not wait for processing which makes the request itself fast and allow more throughput
  - the results aren't returned yet and the connection is held open which consumes memory primarily (but also CPU for keep-alives which is generally kept at 15s)
- If database or in-memory
  - a background async worker polls the database at some defined interval to process all entered requests, preferably in bulk
    - the worker is probably shared across instances if database is shared and per-node if the data is stored in-memory
- If queue
  - the entries are published into queue and consumers process them in a decoupled manner
- Once processing is done, results can be published into a queue again which the writer service instances would be listening to
  - we can use keys to send this event to a specific partition and guarantee being received by the instance that holds the SSE connection
  - the writer instance then sends back the result in the open connection
  - the client closes the SSE connection to free up resources on server
- Open connections consume memory during processing time but no wasted processing for result check

## Comparison

- Client-side polling scales more easily than Event-Driven SSE, and therefore better in high concurrency situations
- Event-Driven SSE prevents repeated result check, and therefore better if write latency cannot be predicted by consumer and is highly variable
