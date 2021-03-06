---
layout:     post
title:      "Building Fault Tolerant Clients With Finagle - Part 1"
comments:   true
tags:
- microservices
- scala
- finagle
---
We've been using [Finagle](http://twitter.github.io/finagle) quite a lot on our current project, so I decided to dig into the [Client](http://twitter.github.io/finagle/guide/Clients.html) implementation in order to check if we are properly using it and understand what else it could offer us.

Finagle currently ships with client implementation for protocols such as **HTTP**, **MySql**, **Redis**, **Memcached**, and is [extensible](https://twitter.github.io/finagle/guide/Extending.html) for any protocol implementation. Initializing a HTTP client for example, would be as simple as:

```scala
val client = Http.client.newService("example.com:80")
```

This creates a new client for the host and port specified, and from that it's fairly simple to send requests out and work with responses as follows:

```scala
val req = Request("/foo", ("my-query-string", "bar"))
req.host = "example.com"

val resp: Future[Response] = client(req)
```

There's also another handy method for retrieving page content based on a HTTP url, which currently doesn't support HTTPS.

```scala
val resp: Future[Response] = Http.fetchUrl("http://example.com")
```

But as we all know, the only certainty we have when integrating with external services is that it will fail at some point in time. That will happen for lots of different reasons such as network problems, bugs on the service, unavailability, etc.

It's really important to understand these types of failures and take them into account when defining clients that will act as integration points with external services.

Currently Finagle client implementation comes with support to **Timeouts**, **Retries**, **Circuit Breakers**, **Tracing**, **Rate Limiting**, among other features, and it can also be customized by composing with custom `Filter` and `Service` implementations.

### Timeouts

Nowadays pretty much every application needs to integrate with other services through the network. In case of any problem with the service or the network, timeouts are used to tell the client when it's time to give up and keep going, instead of waiting for a response that might never come.

The [TimeoutFilter](https://github.com/twitter/finagle/blob/master/finagle-core/src/main/scala/com/twitter/finagle/service/TimeoutFilter.scala) are used to configure global timeouts, that will be applied to every request performed by the client.

```scala
def timeoutFilter[Req, Rep](duration: Duration) = {
  val timer = DefaultTimer.twitter
  new TimeoutFilter[Req, Rep](duration, timer)
}
```

```scala
val clientWithTimeout = timeoutFilter(1.second) andThen client
```

But it's also possible to specify the timeout criteria on request level, which will take precedence over the global timeout configuration.

```scala
client(req) within 1.second
```

### Retries

When a call to a service results on failure, timeout or is successful but returned an undesired result, there's the chance of retrying the request on a hope for a successful and desired result.

The [RetryFilter](https://github.com/twitter/finagle/blob/master/finagle-core/src/main/scala/com/twitter/finagle/service/RetryFilter.scala), when hooked into the client stack, acts coordinating the retries of service executions. By default clients will retry any request that doesn't write anything to the output or throws any kind of `WriteException`. But the retry behavior can be easily customized and applied to pretty much any situation.

The conditions for whether a retry will happen or not, as well as the total number of retries and the time interval between them are defined on the [RetryPolicy](https://github.com/twitter/finagle/blob/master/finagle-core/src/main/scala/com/twitter/finagle/service/RetryPolicy.scala) class.

```scala
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

The code above defines a retry filter, which will retry every request that fails with `TooManyrequestsException`, and will do it for at most 3 times, including the first attempt. The `.tries` factory method uses [jittered back-offs](http://www.awsarchitectureblog.com/2015/03/backoff.html) between retries.

Finagle also allows you to specify your own back-off strategy, in case you need more control over the time interval between executions.

```scala
val backoffs = Stream(1.second, 4.seconds, 8.seconds)
val retryPolicy = RetryPolicy.backoff(backoffs)(retryCondition)
```

#### a note on service retries

As I mentioned earlier, most common causes of service failures involves problems in the network or on the remote system, which won't be resolved right away. So be careful when defining back-off strategies and try not to use immediate retries, as they are likely to fail again for the exactly same reason.

On the other hand, using longer intervals might as well bring undesired results, specially from clients perspective. They will have to wait longer for service responses, risking pushing response time past timeout configuration.

[Michael Nygard](http://www.michaelnygard.com) has an excellent [book](https://www.goodreads.com/book/show/1069827.Release_It_) on the subject where he recommends the queue-and-retry strategy for a slow retry, responding something to the user right away. The goal is to ensure that once the remote service is healthy again the overall system will be able to recover.


### Rate Limited Services

Another important aspect to take into account is rate limiting. It has been widely adopted by most API vendors on the market, and is indeed a good practice to place them in front of services, specially those that will be exposed to the outside world. Rate limiting prevents brute force attacks and is also used to control access quotas for different client profiles.

Recently I was working integrating with a service that only allowed my client key to perform 10 requests within a second. So I wanted to find a way to make the Finagle client coordinate the calls to this API, and avoid as much as I could the propagation of any `HTTP 429 Too Many Requests` errors to end users.

Looking at Finagle codebase I came across the [RequestSemaphoreFilter](https://github.com/twitter/finagle/blob/master/finagle-core/src/main/scala/com/twitter/finagle/filter/RequestSemaphoreFilter.scala) implementation which uses Twitter's [AsyncSemaphore](https://github.com/twitter/util/blob/master/util-core/src/main/scala/com/twitter/concurrent/AsyncSemaphore.scala) to grant permits and enqueue waiters when no more permits are available. That was roughly what I was looking for, except for the fact that it doesn't take time into account.

[Moses Nakamura](https://twitter.com/mnnakamura) introduced me to the recently implemented [AsyncMeter](https://github.com/twitter/util/blob/master/util-core/src/main/scala/com/twitter/concurrent/AsyncMeter.scala), which has a similar behavior to the semaphore implementation, but adding time into the game. I could easily configure a new meter which releases 1 new permit every 100 milliseconds (10 per second), and allows at most 1000 waiters to be enqueued, as demonstrated on the snippet below.

```scala
val meter = AsyncMeter.newMeter(1, 100.milliseconds, 1000)
```
There's also the handy `.perSecond` method, which simplifies this configuration and makes the code more readable.

```scala
val meter = AsyncMeter.perSecond(10, 1000)
```

From that I decided to make my first contribution to the project and implemented the [RequestMeterFilter](https://github.com/twitter/finagle/blob/develop/finagle-core/src/main/scala/com/twitter/finagle/filter/RequestMeterFilter.scala) using the `AsyncMeter`. And as the docs states:

> A `Filter` that rate limits requests to a fixed rate over time by using the `AsyncMeter` implementation. It can be used for slowing down access to throttled resources. Requests that cannot be enqueued to await a permit are failed immediately with a `Failure` that signals that the work was never done, so it's safe to re-enqueue.

Hooking the filter to the client definition is just a matter of instantiating it and adding to the stack, as follows.

```scala
val requestMeter = new RequestMeterFilter(meter)

val limitedClient = requestMeter andThen client

```

This filter will execute a new request every 100 milliseconds and will allow 1000 requests to enqueue, waiting for their time to execute. Once the waiting queue reaches the max size, new requests will be dropped immediately.

On the next post I will continue exploring Finagle client features in greater detail, more specifically Circuit Breakers (Failure Accrual and Fail Fast), Monitoring and client Statistics. But the material presented here is enough to start building fault tolerant clients. Have fun!
