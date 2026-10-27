# Read/Write strategies for high write throughputs

- Regular read/writes may end up being slow for very high throughputs
- In such cases, we can use CQRS to speed up the write path
  -  CQRS is command query responsibility segregation
- If using CQRS, the writer and reader are decoupled and the reader doesn't know when the write operation is actually done
- So if you have to return the result of a processed write while using CQRS, you have following ways to handle it
  - `client-side polling` at some interval to check in a shared database if the result is ready or not and return it [HOW]
  - `event-driven SSE` where the service holds open SSE connections until an event notifies with result and then returns it [HOW]

- Also cover how to handle when data stored per-node [TODO]
