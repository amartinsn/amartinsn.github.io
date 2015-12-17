---
layout:     post
title:      "Building Fault Tolerant Clients With Finagle"
comments:   true
tags:
- microservices
- scala
- finagle
---

We've been using Finagle quite a lot on our current project, and I spent the last few days digging into the `Client` implementation in order to check if we were properly using it and understand what else could it offer us.

Current `Client` implementation comes with support to **Tracing**, **Error Handling**, **Circuit Breakers**, **Monitoring**, among other features, and can be customized be composing it with other `Filters` and `Services`.

Finagle currently ships with Client implementation for protocols such as **HTTP**, **MySql**, **Redis**, **Memcached**, and it allows you to [implement your own](https://twitter.github.io/finagle/guide/Extending.html) Client, for any kind of protocol. Initializing a HTTP client for example, would be as simple as:

```scala
val client = Http.newClient("google.com:80")
```
This creates a new client for the Google website, and from there, you would be able to easily submit new requests and work with resuls as exposed:

```scala
val req = Request("/foo", ("param", "bar"))
req.host = "google.com"

val resp: Future[Response] = client(req)

```
Pretty straightforward, isn't? But as you might already know, integrating with other services, specially those out of your control, requires some preventions.you should be prepared for eventual failures on the client execution, due to a variety of reasons some of them exposed on this post.

### Timeouts

Nowadays pretty much every application deals with networks, which as we all know are fallible for lots of different reasons. When integrating with other systems through the network, it's important to make use of timeouts to prevent your application from waiting forever for a response that might never come. It tells the client when it's time to give up.

Finagle offers a `TimeoutFilter` implementation, which makes this process quite simple.

Finagle allows you to define timeouts globally for every request by using the `TimeoutFilter`.

```scala
def timeoutFilter[Req, Rep](duration: Duration) = {
  val timer = DefaultTimer.twitter
  new TimeoutFilter[Req, Rep](duration, timer)
}
```
Then we can create a HTTP client with support for timeouts, just by hooking the filter to the existing client instance.

```scala
val clientWithTimeout = timeoutFilter(1.second) andThen client
```

Or we could also specify the timeout criteria per request.

```scala
client(req) within 1.second
```

### Retries


### Tracing


### Circuit Breakers



### Initializing Clients

Finagle clients interfaces are exposed, by convention, by objects named after the protocol implementations. For the HTTP protocol, the syntax for creating a new client would be something like:

```scala
Http.newClient("api-host:port")
```

There are also client implementation of other protocols such as Redis, Mysql, Memcached, etc.

```scala
Redis.newClient(...)
Mysql.newClient(...)
...
```

One of the things I like the most on Finagle is that it allows you to compose functionalities by implementing the Pipes and Filters pattern, which enables you to create a much decoupled design, where each `Filter` and `Service` has its own specific responsibility.


It seems simple but there are some considerations you should take into account when implementing clients for services you don't control and I will highlight some of them by sharing some lessons learned from this experience.

### Expect systems to fail

It's crucial to keep in mind when integrating with systems you don't control that it will eventually fail, and you should be ready for when that happens. Finagle currently ships with some interesting filter implementations, one of them is the `RetryFilter`, which handles exactly the case of a service failure. It acts coordinating retries of service executions that returned either successful or error (exceptions) responses. The rules for defining whether a retry should happen or not are defined by the `RetryPolicy` class.

```
val retryCondition: PartialFunction[Try[Nothing], Boolean] = {
  case Throw(WriteException(_)) => true
  case _ => false
}

val retryPolicy = RetryPolicy.tries(3, retryCondition)

val retryFilter = new RetryExceptionsFilter[Request, Response](
   retryPolicy, timer, statsReceiver)
```

The example above creates a policy that will retry any request that throws an exception wrapped in a `WriteException`, and will repeat it for at most 3 times. `Timer` is used to schedule retries and `StatsReceiver` keeps track of the total number of retry occurrences.  

Backoff is another strategy for defining retry policies, giving you more control over the durations bewtween retries.

```
val backoffs = Stream(1.second, 4.seconds, 8.seconds)
val retryPolicy = RetryPolicy.backoff(backoffs)(retryCondition)
```

First retry will happen after 1 second, if it fails again, the client will retry again in 4 seconds, and so on...


### Dealing with rate limiting

Another important aspect to take into account when dealing with external services is rate limiting. Its main purpose is to prevent services from suffering brute force attacks, but it's also widely used on comercial APIs to define pricing plans with access quotas. Independent on the reason, clients must be properly implemented to avoid spanning `HTTP 429 Too Many Requests` errors to users.

Finagle offers a filter called `RateLimitingFilter` where you can specify the amount of requests allowed for a given client key, during a time period. If client rate goes over the limit, filter start to drop new requests, and might eventually open circuit.

```
val clientKey = "foo" // something that identifies a client or a group
val strategy = new LocalRateLimitingStrategy[Int](clientId, 1.second, 5)
val filter = new RateLimitingFilter[Int, Int](strategy)
```

On the other way, it's also important for your client implementation to know how to deal with rate limited services. Let's say you are consumming an API and your key only allows no more than 8 requests per second to this service. You could either send `HTTP 429 Too Many Requests` responses back to those using the client or configure your client coordinate requests to services based on the rate limit defined for your key.

Having a chat with Moses Nakamura, one of Finagle developers, he mentioned me to `AsyncMeter`, a feature still under development on Twitter util project.

Of course you have the option of sending `HTTP 429 Too Many Requests` to users but it would be much less invasive if API client could at least try to handle it itself. Chatting with Moses Nakamura, one of Finagle devs, he mentioned they were working on a util class called `AsyncMeter`.


### Circuit breakers


### Final code

```
implicit val timer = DefaultTimer.twitter
val statsReceiver = new MetricsStatsReceiver()

private val retryFilter = {
  val retryCondition: PartialFunction[Try[Nothing], Boolean] = {
    case Throw(WriteException(_)) => true
    case _ => false
  }

  new RetryExceptionsFilter[Request, Response](
    RetryPolicy.tries(3, retryCondition), timer, statsReceiver)
}

private val throttler = TimerBasedThrottlerFilter[Request, Response](
	1, 150 milliseconds)(timer)

private val httpClient = Http.client.newService("the-api-host:port")

// client composed with all filters defined
val defaultClient = throttler andThen retryFilter andThen httpClient

```
