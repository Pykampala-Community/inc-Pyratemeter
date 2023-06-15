``Pyratemeter` is a Python based rate limiter tool used in a network system to control the rate of 
traffic sent by a client or a service preferably over an http network. Therefore, Pyratemeter
is used to limit the number of client requests allowed to be sent over a specified period in
a way that if the API request count exceeds the threshold defined by Pyratemeter, all the
excess calls are blocked.

Why is it necessary to build Pyratemeter?
-----------------------------------------

1. Prevent resource starvation caused by Denial of Service (DoS) attack. Almost all APIs 
   published by large tech companies such as Twitter, Google and many others enforce some form of rate limiting. Therfore, a Pyratemeter prevents DoS attacks by blocking the excess calls

2. Reduce cost. Limiting excess requests means fewer servers and allocating more resources to high prority
   APIs. For this reason, rate limiting is extremely important for companies that use paid third party
   APIs. For example, you are charged on a per-call basis for the following external APIs: check credit,
   make a payment, retrieve health records etc.

3. Prevent servers from being overloaded. To reduce server load, Pyratemeter is used to filter out excess
   requests caused by bots or users' misbehavior

Low-level design for Pyratemeter
----------------------------------

Considering a client-server communication, intuitively we can implement rate limiting using Pyratemeter
at either the client side or server side however, generally speaking client side is an unreliable place
to enforce rate limiting because client  requests can easily be forged by malicious actors and greatly 
we don't really have control over client implementation.So placing our Pyratemeter at the server-side is quite okay however it's realy not ideal. Forexample:

                                             ______________________________
     CLIENT -------------------------------> |                Pyratemeter |
                   HTTP request              | API serevers               |
                                             |                            |
                                              _____________________________

Therefore, Pyratemeter being a stand alone tool, it's not a good idea to place it at the API servers. So instead of achitecting the Pyratemeter to be used on the server-side, we create kinder of a middleware
which throttles requests to server APIs as illustrated below.

                                      |
                                      |
                                      |
      CLIENT------------------------->|-----------------------> API SERVERS
                                      |
                                      |
                                      |
                                   Pyratemeter

Explanation:
------------
           Assume our API allows 2 requests per-second and the client sends 3 requests to the server
           within a second. The first two requests get routed to the API servers however the Pyratemeter
           throttles the third request and returnd an HTTP status code 429. The HTTP 429 response status
           code indicates a user has sent too many requests as illustrated below.
                                     
                                           Pyratemeter
                                               |
                    -----req-1---------------->|
            CLIENT  -----req-2---------------->|-------------------------> API SERVERS
                    -----req-3 throttled------>|------------------------->
                    <--------------------------|
                      429:Too many requests
                    (Throttling by Pyratemeter)

Cloud microservices have become widely popular and rate limiting is usually implemented within a 
component called API gateway. An API gateway in this case is a fully managed service that supports rate 
limiting, SSL termination, authentication, IP whitelisting, serving static content etc. For this reason
knowing that API gateways support rate limiting, it's a design decision that the Pyratemeter is designed 
in a way that it's able to support rate limiting both on the server-side and on API gateways.

Algorithm for Pyratemeter
--------------------------
Pyratemeter uses the `Token Bucket` algorithm for throttling API requests.

`How the Token Bucket algorithm works`:

   - A `token bucket` is a container that uses `pre-defined` capacity. Tokens are put in the bucket
     at preset rates periodically. Once the bucket is full, no more tokens are added.
     For example, consider a token bucket of capacity 4. The refiller puts 2 tokens into the bucket
     every second and once the bucket is full, extra tokens will overflow.
    
   - Each request consumes one token. When a request arrives, we check if there are enough tokens in 
     the bucket. If this is true, we take one token out for each request, and the request goes through.
     If there are not enough tokens, the request is dropped.
    
`The token bucket algorithm takes two parameters:`
   
   - Bucket size: the maximum number of tokens allowed in the bucket

   - Refill rate: number of tokens put into the bucket every second

`Pros of the token bucket algorithm`

   - The algorithm is easy to implement

   - Memory efficient

   - Token bucket allows a burst of traffic for short periods. A request can go through as long as
     there are tokens left

`cons of the token bucket algorithm`

  - Two parameters in the algorithm are bucket size and token refill rate. However, it might be 
    challenging to tune them properly

High-level architecture of the Pyratemeter
--------------------------------------------
The basic idea of the Pyratemeter algorithm is, at the high-level we need a counter to keep track 
of how many requests are sent from the same user, IP address etc. If the counter is larger than
the limit, the request is disallowed.

We don't store the counters using the database due to slowness of disk access. `In-memory cache` is 
chosen because it is fast and supports time-based expirations strategy. Therefore, it is in memory
store that offers two commands:`INCR and EXPIRE`

   - `INCR`: It increases the stored counter by 1

   - `EXPIRE`: It sets a timeout for the counter. If the timeout expires, the counter is automatically
               deleted

illustration for the high-level architecture
--------------------------------------------

First, the client sends a request to Pyratemeter middleware

Pyratemeter middleware then fetches the counter from the coressponding bucket in the in-memory cache
such as (Redis) and checks if the limit reached or not. If the limit is reached, the request is rejected
otherwise the request is sent to API servers. Meanwhile the system increments the counter and save it
back to in-memory cache 

                         |
                         |----------->API SERVERS
        CLIENT---------->|                 
                         |<---------->IN-MEMORY CACHE
                         |         
                 Pyratemeter middlware       



Exceeding rate limit and rate limit headers
-------------------------------------------
In case a request is rate limited, APIs return an HTTP response code 429(too many request)to the client.
Depending on the use-case, we enqueu the rate-limited requests to be processed later. For example, if some
requests are rate-limited due to system overload, the Pyratemeter keeps those requests to be processed later.

How the client knows whether it is being throttled and the number of allowed remaining requests before
being throttled will depends on the HTTP response headers. The Pyratemeter returns the following 
HTTP response headers to clients.
  
  1. X-Ratelimit-Remaining: The remaining number of allowed requests within the time window.

  2. X-Ratelimit-Limit: It indicates how many calls the client can make per time window.

  3. X-Ratelimit-Retry-After: The number of seconds to wait until you can make a request again.

So, when a user has sent too many requests, a 429 too many requests error and X-Ratelimit-Retry-After header
are returned to the client.

Capability to work in distributed environments.
------------------------------------------------
Higly Pyratemeter to be able to work in a distributed environment to support multiple servers and concurrent
threads, we address two issues.
  
  1. Race condition issue
     Race conditions can happen in a highly concurrent environment. let us assume the counter value
     in in-memory cache is 3. If the two requests concurrently read the counter value before either
     of them writes the value back, each will increment the counter by one and write it back without
     checking the other thread. Both requests(threads) believe they have the correct counter value is 4.
     However the correct counter value should be 5.
     
     So, locks are used to solve this issue however they significantly slow down the system. Therefore, 
     two strategies are used ie `Lua script` and `sorted sets data structure`
    
  2. Synchronization issue
     Synchronization is another issue to consider in a hihly distributed system. To support millions of 
     users there must be a mechanism to handle traffic from more than one client (multiple clients)

Performance optimization
-------------------------
`Multi-data center` setup mechanism is crucial for Pyratemeter because `latency` is high for users located 
far away from the data center. Most cloud service providers build many edge server locations around the world. 
In such a case, traffic will be automatically routed to the closest edge server to `reduce latency` and lastly synchronization of data with an eventual consistency model.

Monitoring
-----------
Gathering analytics is a key feature Pyratemeter adopts to check whether
   
   - The rate limiting algorithm is effective

   - The rate limiting rules are effective

------------------------------------------------------------------------------------------------------------ 

## Thanks For Reading Throuh, Happy Hacking !