# Review Feedback Analysis [TODO]

## Things you could have done better

1. Thought of distributed vs single instance complexities when deciding hash functions
2. Thought of Resharding up front so that you could have avoided reworking the entire solution approach
3. Done the math at each step starting from actual pain point
   1. happened before peer-review where we started sequence reuse before looking at if that will ever be required
   2. happened after peer-review again when insert batching was a possible solution but we didn't check if batching on a single server instance would ever have enough items in batch to make a difference
4. Just ended up assuming that SSE would work across multiple services whereas it doesn't work just like that and needed some discovery after all
