---
layout:     post
title:      "Building Fault Tolerant Clients With Finagle - Part 2"
comments:   true
tags:
- microservices
- scala
- finagle
---
When comparing the Finagle Failure Accrual to the Hystrix Circuit Breaker I conclude that the main difference is in how they are configured. In Finagle one configures absolutely the number of consecutive failures that will trigger fail-fast, whereas with Hystrix a percentage is used together with a minimum number of requests that have to occurred inside a specified timeframe before the the circuit breaker will be opened.

What are the pros and cons of both solutions? It appears that the Finagle version is more rigid and would therefore only work in situations where services are up a lot of the time, or am I mistaken?

Cheers,
Alessandro

======================

I think the main advantage of the finagle failure accrual implementation is that it's dirt-cheap.  It also assumes that you have a very low error rate normally–we typically assume our success rate will be very near 100%.  If your typical SR is 99.9%, and we assume failures are uniformly distributed (they're not) then you only get a false-positives for 5 failures in a row (1 - 0.001^5) * 100 % of the time.  You will also get true-positives very fast, and false-negatives seem not terribly likely (you'd have to have a failure pattern where some requests always fail and some requests always succeed, and either very low throughput or a very, very regular pattern).

Taking a quick look at the hystrix circuit breaker implementation, it looks like it uses windowing, where a counts success vs failure, and then moves over the window every "duration".  It gets around the "my new window might not be representative because I have too few samples" problem by periodically taking a snapshot of several windows, and it seems that the default snapshot size is 500ms.  So it sounds like it generally won't detect a problem in faster than 500ms, (although this is configurable) and it may not respond in time to a problem where all of your requests fail for 50ms.

So, to sum up:

The finagle implementation is simple, and is optimized for detecting problems immediately, with a low false-positive rate, and very low false-negative rate, when you're working under the assumption that your services see very few failures.

The hystrix implementation seems a little more sophisticated, and is optimized for having very high accuracy and precision.  It might take a little while to trip the circuit breaker, and has relatively low resolution by default.

TL;DR yes, you're right, the finagle failure accrual won't work as well if your services tend to have a very low success rate during normal operation.  I'd be curious about why you expect your services to have such low success rates.

========================

Finagle's solution is also more sensitive to bursty failures. In many real-life workloads, arrival of requests are often far from random, and there are artifacts (chunking, for example) that further contributes to the spikiness of requests, leading to a higher false-positive ratio than most theoretical models suggest.

Agree that FailureAccrual is the more straightforward mechanism and cheaper implementation.

Finagle's other assumption is it is relatively cheap/feasible to avoid the flagged endpoint/client, that an alternative route is readily available and just as good. A more accurate failure detection, even more expensive, can be justified or desirable if the cost of load balancing is higher or not quite as desirable as the default route.

-Yao (@thinkingfish)



============
other thread
============

When making requests using finagle client, except timeout exception, we also find some FailedFastException:

ERR [20130413-06:31:09.130] logtorrent: ====UMID Exception:class com.twitter.finagle.FailedFastException,msg: null,Stacktrace begin
ERR [20130413-06:31:09.130] logtorrent: com.twitter.finagle.NoStacktrace(Unknown Source)
ERR [20130413-06:31:09.130] logtorrent: ====End

 Except for the exception class name, no more info is available,

 I notice that ‘fail fast’ has something to do with the service availability checking,but can anybody explain more details?

PS.In our case, we use the finagle client to issue requests to another service, when we detect that service is down(or temporarily unavailable) we want to skip it and sometime later re-use it if the service comes back.

now we implement the logic by ourself, but seems that finagle client can do this for me(mark a host as down and retry later), if yes,can more details be given, like the params to control the interval to retry?

========================================

With fail fast, a connection will be marked down if a request to it fails. On failure the connection will be marked as unavailable and a background thread will be spun up to attempt reconnection via a backoff retry policy. The default policy is to use an exponential backoff of 1 second that grows by a factor of 2 with a max of 32 seconds.

You can change the policy by setting failFast to false then wrapping your built factory in your own FailFastFactory instance.

========================================

This being said—you get FailFastExceptions only when *all* hosts have entered fail-fast mode (maybe you have only one host in your cluster).
