I spent last week playing with Finagle Client to understand what it has to offer and how is that compared to other clients. Finagle currently offers robust clients for protocol implementations such as HTTP, MySql, Redis, Memcached, etc. I decided to pick HTTP client and explore things I could do with it.

The idea was implement a robust client for a public API, with authentication and rate limiting. And the goal for this post is to share some lessons learned from this experience.

### Initializing Clients

Finagle clients interfaces are exposed, by convention, on objects named after the protocol implementations. So for example, if you wanted to create a new HTTP client, the syntax would be something like:

```
Http.newClient("api-host:port")
```
or for other protocols:

```
Redis.newClient(...)
Mysql.newClient(...)
...
```

One of the things I like the most on Finagle is that it allows you to compose functionalities by implementing Pipes and Filters pattern, which enables you to create a much decoupled design, where each `Filter` and `Service` has its own specific responsibility.

### Expect failure from systems you can't control

The most important thing to keep in mind when integrating to 3rd party systems is that it will fail at some point in time, and you should be ready for when that happens. Finagle currently ships with some very interesting filter implementations. One of them is the `RetryFilter`, which coordinates retries of service executions. Both successful or exceptional responses are retryable and the rules that defines whether a request should be retried or not are defined on the `RetryPolicy`.

```
val retryCondition: PartialFunction[Try[Nothing], Boolean] = {
  case Throw(WriteException(_)) => true
  case _ => false
}

val retryPolicy = RetryPolicy.tries(3, retryCondition)

val retryFilter = new RetryExceptionsFilter[Request, Response](
   retryPolicy, timer, statsReceiver)
```
The example above creates a policy that reties any request that throws any `Exception` wrapped on a `WriteException`. It will retry the request for at most 3 times. `Timer` is used to schedule retries and `StatsReceiver` keeps track of the number of retry occurrences.  

Backoff is another option for defining retry policies. That way you have more control and flexibility on the durations bwtween retries.

```
val backoffs = Stream(1.second, 2.seconds, 3.seconds)
val retryPolicy = RetryPolicy.backoff(backoffs)(retryCondition)
```


### Dealing with rate limiting

Another important aspect to consider when shipping services that will be exposed to the outside world, or even to your company is rate limiting. It prevents users of your services to flood your server with consecutive calls, which might impact response time and availability. So it's strongly recommended to add it to your service stack. Finagle

```
val strategy = new LocalRateLimitingStrategy[Int](categorize, 1.second, 5)
val filter = new RateLimitingFilter[Int, Int](strategy)
```

On the other way, it's also important to tell your client how to consume rate limited services. Let's say you are consumming an API and your api key only allows you to make 8 requests per second. Of course you have the option of sending `HTTP 429 Too Many Requests` to users but it would be much less invasive if API client could at least try to handle it itself. Chatting with Moses Nakamura, one of Finagle devs, he mentioned they were working on a util class called `AsyncMeter`.
