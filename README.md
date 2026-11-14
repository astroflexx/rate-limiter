# rate-limiter
# a distributed rate limiter

- a rate limiter is one of the most important components of a distributed systems architecture, crucial for protecting resources from abuse
- something similar to Cloudflare / Envoy
- Cloudflare and Stripe's rate limiters are insane
- lets write this project in Java

# importance / need

- protect resources from abuse. prevent DDoS attacks, prevent resources like API servers and such from being overwhelmed or abused.
- reduce cost. prevent excess usage of expensive APIs especially when calling paid third-party APIs to save money

# functional and non-functional requirements

## functional

- accurately limit excessive requests
- multi-window support. have multiple rules at different time granularity (like per second, per hour, per day)
for the same resource, so that even if one rule gets bypassed the others still limit requests
- exception handling. show users when they are rate limited, with helpful headers like try-again-after or
also inform them what the limit is

## non-functional

- low latency. the rate limiter must not slow down requests
- strong consistency. it should work in a distributed systems environment, with multiple servers

# numbers


# standard algorithms

## 1. token bucket

- have a bucket with tokens, with a max capacity
- every request consumes / requires some number of tokens
- a refiller refills the bucket with tokens every fixed interval, making sure that it never exceeds the capacity of the bucket
<br><br>
- this allows a burst of requests to go thru, if the bucket has enough tokens for it
- so the service handling these requests can handle a burst of them at a time, or one at a time, or at any rate
<br><br>
- we need to atomically subtract tokens when many requests come in
- and the refiller also needs to atomically add tokens
- we usually need a `(request need, capacity, refill rate)` per user per API endpoint
- so these 3 params need to be set carefully
- realistically, most implementations assume 1 token per request, so theres mainly 2 params to tune here

## 2. leaking bucket

- have a queue with a fixed capacity
- when a request comes in, check the size of the queue
- if enough space, push the request into the queue. else reject the request
- the service can then process the requests from the queue at a fixed / steady rate
<br><br>
- this algorithm is mainly used when we want a steady rate of requests to be processed,
rather than varying rates
- token bucket does not guarantee a fixed rate of requests, and since most services like databases
are designed to handle a fixed rate of reads / writes per second and won't be able to absorb traffic spikes,
leaky bucket helps to provide a steady constant rate of requests to services to process

## 3. fixed window

- have a fixed window size of time, say 1 second or 30 seconds or whatever
- have a max count of requests for the time window, say like 30
- so just maintain the count of requests for the time window, and reject any requests that exceeds the count
<br><br>
- the problem here is at edges, you can send x requests at the end of a window,
and another x requests at the start of the next window, and thus you were able to make
2x requests in the last one time window
- since this algorithm follows windows at fixed time points, not a general window that can slide over the time scale. 
thus this disadvantage exists

## 4. sliding window log

- we first define the size of the time window, say 10 seconds
- maintain a deque of timestamps of the requests that came in and were accepted
- when a new request comes in, first `pop_front()` all requests whose timestamps are older than `now - window_size`
- then check if queue has space for the  new request. if yes, push the timestamp of the current request. else reject the request
<br><br>
- this algorithm is very accurate. it always ensures requests never exceed the limit within any time window
- however, storing such timestamps is expensive and it consumes a lot of memory

## 5. sliding window counter

- use the same fixed windows as before
- have a max count for each window
<br><br>
- say the limit is 7 requests per time window of 1 minute
- when a new request comes in, say 30% of the current time window has elapsed
- and 3 requests have been processed in the current time window so far
- so now we approximate how many requests have been processed in the last 70% of the previous window
- which is 70% of the total count of the previous window
- say 5 requests in total were processed in the previous window. so 70% of 5 is 3.5
- that gives a total of 6.5 requests in the last window, we can round this to 6
- and since 6 < 7, the current request is allowed
<br><br>
- obviously here, we assumed the 5 requests in the previous window were distributed evenly in the window
- that is the ideal situation or desired state, however it doesn't always happen so
- if all 5 requests were at the start of the previous window, we could rate limit a legitimate request
- or if all 5 requests were at the end of the previous window, we could allow a bad request
- so this is an obvious problem here
- this is Cloudflare's approach to sliding window counter
<br><br>
- another approach is, lets say the rate limit is 100 req per hour
- so the main window size is 1 hour
- we can divide that into subwindows of some size that divides the main size, say 1 minute
- so there would be 60 subwindows in a single main window
- and now we store a count of no of requests in every subwindow
- in a deque, we maintain only the last subwindows such that the subwindow's start time >= now - main window size
- `pop_front()` older entries
- we also maintain a sum of the counts of these subwindows
- and increment this sum when a new request is allowed, or subtract when an old window is popped
<br><br>
- now when a new request comes in, we check this sum, which is the sum of the current valid subwindows' counts
- check if the sum is < limit, and increment it, and also increment the current subwindow's count
- this algorithm is basically a combination of fixed window and sliding window log
- it neatly balances memory but also provides reasonable accuracy
- not 100% accurate since the oldest bucket could have more or less requests outside the subwindow time, which could
allow bad requests or deny good requests
- but the error margin is only upto one subwindow size, which could be acceptable
- this is Figma's approach to sliding window counter

<hr>

