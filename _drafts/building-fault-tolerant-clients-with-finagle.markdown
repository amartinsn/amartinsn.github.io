---
layout:     post
title:      "Building Fault Tolerant Clients With Finagle"
comments:   true
tags:
- microservices
- scala
- finagle
---
We've been using Finagle quite a lot on our current project, and I spent the last few days digging into the `Client` implementation in order to check if we are properly using it and understand what else it could do for us.

Finagle currently ships with client implementation for protocols such as **HTTP**, **MySql**, **Redis**, **Memcached**, and is [extensible](https://twitter.github.io/finagle/guide/Extending.html) for any protocol implementation. Initializing a HTTP client for example, would be as simple as:

```
val client = Http.newClient("google.com:80")
```

This creates a new client for the Google website, and from that it's fairly easy to send requests to it and work with responses as follows:

```
val req = Request("/foo", ("param", "bar"))
req.host = "google.com"

val resp: Future[Response] = client(req)

```

But as we all know, the only certainty we have when integrating with external services is that it will fail at some point in time. It will happen for lots of different reasons such as network problems, bugs on the service, unavailability, etc.

It's really important to understand these types of failures and take them into account when defining clients that will act as integration points with these services.

Currently Finalge client implementation comes with support to **Timeouts**, **Retries**, **Circuit Breakers**, **Tracing**, among other features. It can also be customized by composing with custom `Filter` and `Service` implementations.

### Timeouts

Nowadays pretty much every application needs to integrate with other services through the network. In case of any problem with the service or the network, timeouts are used to tell the client when it's time to give up and keep going, instead of waiting for a response that might never come.

The `TimeoutFilter` are used to configure global timeouts, that will be applied to every request performed by the client.

```
def timeoutFilter[Req, Rep](duration: Duration) = {
  val timer = DefaultTimer.twitter
  new TimeoutFilter[Req, Rep](duration, timer)
}
```

```
val clientWithTimeout = timeoutFilter(1.second) andThen client
```

It's also possible to specify the timeout criteria on request level. That will take precedence over the global timeout configuration.

```
client(req) within 1.second
```

### Retries

Calls to external services might fail for several reasons and the `RetryFilter`, when hooked into the client, will be responsible for coordinating the retries on these service executions. Retries are not restricted to failed responses, it's also possible to retry successful responses.

By default clients will retry any request that doesn't write anything to the output or throws any kind of `WriteException`. But the retry behavior is easily extensible  and can be applied to pretty much any situation.

The rules for defining whether a retry should happen or not, as well as the total number of retries and the time interval between them are defined on the `RetryPolicy` class.

```
val retryCondition: PartialFunction[Try[Nothing], Boolean] = {
  case Throw(error) => error match {
    case e: TooManyRequestsException => true
    case _ => false
  }
  case _ => false
}

val retryPolicy = RetryPolicy.tries(3, retryCondition)

val retryFilter = new RetryExceptionsFilter[Request, Response](
   retryPolicy, timer, statsReceiver)
```

The code above defines a retry filter, which will retry every request that fails with `TooManyrequestsException`, and will do it for at most 3 times, including the first attempt. The `.tries` factory method uses [jittered backoffs](http://www.awsarchitectureblog.com/2015/03/backoff.html) between retries.

Finagle also allows you to specify your own backoff strategy, in case you need more control over the time interval between executions.

```
val backoffs = Stream(1.second, 4.seconds, 8.seconds)
val retryPolicy = RetryPolicy.backoff(backoffs)(retryCondition)
```

#### a note on service retries

Most common causes of service failures involves problems in the network or on the remote system, which won't be resolved right away. So be careful when defining backoffs strategies and try not to use imediate retries, as they are likely to fail again for the exactly same reason.

On the other hand, using longer intervals might as well bring undesired results, specialy from clients perspective. They will have to wait longer for service responses, risking pushing response time past timeout configuration.

[Michael Nygard](http://www.michaelnygard.com) has an excelent [book](https://www.goodreads.com/book/show/1069827.Release_It_) on the subject where he recommends the queue-and-retry strategy for a slow retry, responding something to the user right away. The goal is to ensure that once the remote service is healthy again the overall system will be able to recover.

### Circuit Breakers

remember that just as failures cascade, so do delays. While you are waiting or retrying, scarce resources are being used (threads, memory, TCPIP ports, database connections) that cannot be used for other requests.

that's why we need a circuit breaker...

### Tracing
TODO


### Rate Limited Services

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
