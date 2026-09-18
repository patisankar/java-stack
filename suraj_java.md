## stream

https://substack.com/home/post/p-180057277

AtomicLong makes the individual updates thread-safe and the total should be correct, but it is not the best design.

I would use mapToLong(...).sum() because it expresses the reduction clearly and allows the stream to combine partial results efficiently.
