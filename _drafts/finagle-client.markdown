---
layout:     post
title:      "Finagle Clients"
comments:   true
tags:
- microservices
- scala
- finagle
---

Last week I spent some time exploring Finagle Client implementation, understanding the underlyin implementation and how it helps building robust fault tolerant client implementations. Finagle already have client implementations for protocols such as HTTP, MySql, Redis and Memcached, and as far as I could see, it's pretty straightforward to define your own. During my experiments I decided to create a simple HTTP client for a public API, and then step-by-step improve it, making it more fault tolerant.

### Initializing Clients

Finagle clients interfaces are exposed, by convention, by objects named after the protocol implementations. So for example, if you wanted to create a new HTTP client, the syntax would be something like:

```
Http.newClient("api-host:port")
```

or for other protocols:

```
Redis.newClient(...)
Mysql.newClient(...)
...
```

One of the things I like the most on Finagle is that it allows you to compose functionalities by implementing the Pipes and Filters pattern, which enables you to create a much decoupled design, where each `Filter` and `Service` has its own specific responsibility.

### Expect systems to fail

It's crucial to keep in mind when integrating to systems out of your control is that it will eventually fail, and you should be ready for when that happens. Finagle currently ships with some interesting filter implementations, one of them is the `RetryFilter`, which handles exactly the case of a service failure. It is responsible for coordinating retries of service executions, both successful or error (exceptions) responses might be retryable. The rules for defining whether it should happen or not are defined by the `RetryPolicy` class.

```
val retryCondition: PartialFunction[Try[Nothing], Boolean] = {
  case Throw(WriteException(_)) => true
  case _ => false
}

val retryPolicy = RetryPolicy.tries(3, retryCondition)

val retryFilter = new RetryExceptionsFilter[Request, Response](
   retryPolicy, timer, statsReceiver)
```

The example above creates a policy that will retry any request that throws an exception wrapped on a `WriteException`, and will do it for at most 3 times. `Timer` is used to schedule retries and `StatsReceiver` keeps track of the total number of retry occurrences.  

Backoff is another option for defining retry policies, giving you more control over the durations bewtween retries.

```
val backoffs = Stream(1.second, 4.seconds, 8.seconds)
val retryPolicy = RetryPolicy.backoff(backoffs)(retryCondition)
```

First retry will happen after 1 second, if it fails again, will retry again in 4 seconds, and so on...


### Dealing with rate limiting

One important aspect to consider when shipping services, specially those that will be exposed to the outside world is rate limiting. It helps you prevent attacks to your services, which might impact response time or even availability. So it's strongly recommended to rate limit client requests. Finagle offers a filter called `RateLimitingFilter` which helps you prevent that. All you need to do is define a strategy, specifying how many requests the service will process during a time period.

```
val clientId = "foo" // something that identifies a client or a group
val strategy = new LocalRateLimitingStrategy[Int](clientId, 1.second, 5)
val filter = new RateLimitingFilter[Int, Int](strategy)
```

On the other way, it's also important for your client implementation to know how to deal with rate limited services. Let's say you are consumming an API and your key only allows no more than 8 requests per second to this service. You could either send `HTTP 429 Too Many Requests` responses back to those using the client or configure your client coordinate requests to services based on the rate limit defined for your key.

Having a chat with Moses Nakamura, one of Finagle developers, he mentioned me to `AsyncMeter`, a feature still under development on Twitter util project.

Of course you have the option of sending `HTTP 429 Too Many Requests` to users but it would be much less invasive if API client could at least try to handle it itself. Chatting with Moses Nakamura, one of Finagle devs, he mentioned they were working on a util class called `AsyncMeter`.


### Circuit breakers
