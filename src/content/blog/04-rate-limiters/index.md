---
title: "Handling traffic with rate limiters"
description: "A TypeScript implementation"
date: "Jul 21 2025"
draft: false
---
Many times systems are overwhelmed by huge amounts of requests. Many of them implement a variety of mitigation strategies: load balancers, asynchronous queue systems, and rate limiters.

The latter controls traffic by denying repeated requests in a given time frame according to a common identifier, such as an IP address or an authentication token.

For instance, if the same address makes too many requests in a short period of time, the rate limiter will act by dropping them.
### The case of PIX

In June 2025, the Brazilian instant payment system (PIX) reached an all-time high of 280 million transactions in a single day.

The Central Bank of Brazil provides an API service (DICT API) mapping keys such as social numbers or e-mail addresses to account details. Thus allowing the payer to confirm the identity of the receiver.

In order to preserve its stability and security, the service enforces a very strict traffic policy through a *token bucket* algorithm.

The actual implementation is more nuanced, with different scopes and categories per client. For full details, refer to the [official documentation](https://www.bcb.gov.br/content/estabilidadefinanceira/pix/API-DICT.html#section/Seguranca/Limitacao-de-requisicoes).

In this article, I’ll focus on a simplified implementation of the **token bucket** algorithm.
## The token bucket logic
A token bucket has:
- A maximum capacity
- The rate at which it is filled by a certain amount
- Tokens representing the ability to perform a unit of work (like a request).

The basic behavior is:
- On receiving a request, check if the bucket has available tokens.
	- If true, **consume one** token and allow the request.
	- If not, **drop the request**.
- At fixed intervals, the bucket will be refilled:
	- If the bucket is full, do nothing
	- If not, refill accordingly

Variations may include queueing requests or allowing burst traffic, but this is the base logic.
### TypeScript implementation
Let's start by defining the bucket's capacity and refill rate.

	These values would typically be loaded from environment variables, but you get the idea.

```ts
const BUCKET_CAPACITY = 5;
const BUCKET_REFILL_RATE = 5000; // in ms
```

Then, let's define the properties of a `TokenBucket` class and assign them in the constructor. In order to uniquely identify a bucket, a `userId` extracted from a JWT payload will do.

Additionally, we require a property to assist us in handling the refill logic. Since we won't be using the kinds of `setInterval()`, let's simply store a timestamp of the last refill.
```ts
export default class TokenBucket {
    public refilledAt: Date;
    public id: string;
    private capacity: number;
    private remaining: number;

    constructor(
        id: string,
        tokens = BUCKET_CAPACITY,
        refilledAt = new Date(),
    ) {
        this.id = id;
        this.remaining = tokens;
        this.refilledAt = refilledAt;
        this.capacity = BUCKET_CAPACITY;
    }
```
Consuming a token or checking the remaining amount is simple:
```ts
    public consumeToken(): void {
        this.refillIfNeeded();
        if (this.remaining > 0) this.remaining -= 1;
    }
    
    public getTokensLeft(): number {
        return this.remaining;
    }
    
    public hasAvailableTokens(): boolean {
        this.refillIfNeeded();
        return this.remaining > 0;
    }
```

Before processing, the bucket checks whether it needs refilling. Since the last refill timestamp is the only information available, the refill logic will have to accommodate:

- Calculating the number of refill intervals passed since the last refill.
- Adding to the remaining amount accordingly, without exceeding the bucket's capacity.  
- Updating `refilledAt` timestamp.
```ts
    private refillIfNeeded(): void {
        const now = new Date();
        const elapsed = now.getTime() - this.refilledAt.getTime();
        if (elapsed < BUCKET_REFILL_RATE) return;
        const intervalsPassed = Math.floor(elapsed / BUCKET_REFILL_RATE);
        if (intervalsPassed <= 0) return;
        this.remaining = Math.min(
            this.capacity,
            this.remaining + intervalsPassed,

        );
        this.refilledAt = new Date(
            this.refilledAt.getTime() +
                intervalsPassed * BUCKET_REFILL_RATE,
        );
    }
}
```
## Storing the buckets

The simplest way to store token buckets is in the service memory. It's a simple matter of pushing and retrieving bucket entities from a data structure of your choice.
```ts
import TokenBucket from "./TokenBucket";

export default class BucketStorage {
    private buckets: Map<string, TokenBucket>;
    
    constructor() {
        this.buckets = new Map();
    }
  
    async getBucket(id: string): Promise<TokenBucket> {
        const bucket = this.buckets.get(id);
        if (bucket) return bucket;
        const newBucket = new LeakyBucket(id);
        this.buckets.set(id, newBucket);
        return newBucket;
    }

    async saveBucket(bucket: TokenBucket): Promise<void> {
        this.buckets.set(bucket.id, bucket);
    }
}
```
The issue with this solution is that the `BucketStorage` will be gone if the application crashes. An external, reliable solution is a better alternative to store such critical data.

Also, remember that, as bucket logic is an intermediate step between the user and the feature, it should only add the minimum necessary latency to not hurt the user experience.

That sounds like the place for an in-memory database such as Redis, which stores data directly in the main memory.

Check my [repository](https://github.com/twillecke/token-bucket--koa) to see this implementation.
## Rate limiting headers
When returning a response from a service protected by rate limiters, it is adequate to provide information about its policy, quota, and current service limits.

```ts
    res.set(
    {
        "X-RateLimit-Limit":  bucket.getLimit().toString());
        "X-RateLimit-Remaining":     bucket.getRemaining().toString();
        "X-RateLimit-Reset": bucket.getResetTime().toString();
    }
```

The semantics of each header vary according to policy and implementation. But all three mentioned are widely adopted. For a deeper insight, check the [IETF draft RFC on rate-limit headers](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/) proposing a more structured and standardized format.
## Wrap-up
- Systems use different strategies to handle traffic, such as load balancers, asynchronous messaging, or rate limiters.
- Token bucket is an algorithm usually implemented in rate limiters, controlling the flow of requests in intervals of time.
- These strategies add another layer of services and latency and must be optimized so as not to harm the user experience.
