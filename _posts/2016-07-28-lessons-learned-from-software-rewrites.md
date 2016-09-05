---
layout:     post
title:      "Lessons Learned From Software Rewrites"
comments:   true
tags:
- architecture
- software design
- refactoring
---
A couple of years ago I joined a team working on one of [Globo](http://globo.com)’s core products, the platform responsible for user identification and authorization within all our products. With the increasing necessity to know more about our users, to be able to offer them a better experience on our products, my main challenge was to help this team improve the current software architecture, which was old and unreliable, impacting the cost of developing new features.

The primary service, called “Version3”, with all functionalities was implemented back in 2004, using Java and the EJB stack. All services talking to it were restricted to use Java, as the only integration point were the EJB clients. As time passed and development teams started creating systems using technologies other than Java, the company felt they needed a better solution, one that was agnostic to any programming language, so they decided to build “Version4”, a feature compatible RESTful implementation of “Version3”.

Pretty much all features were rewritten on the new stack, some of them didn’t even make sense to the business anymore, but due to the lack of service usage metrics, they were also added to the scope, a massive waste of development effort.

With the release of the new HTTP implementation, most of the clients started using it right away, but a few other ones ended up continuing using the EJB implementation, as the cost of migrating all integrations points from EJB to HTTP was quite high. As a result, we ended up with two implementations of the same set of functionalities.

![](/assets/article_images/lessons-learned-from-software-rewrites/initial-scenario.png)

This approach worked pretty well for some time until requests to new features started to pop out. The team had to implement them on both versions for compatibility, and those who have worked with EJB knows how hard and costly it is to test implementations, especially running acceptance tests when you need to spin up the container which is quite slow. That was affecting considerably the team’s ability to extend the system.

###Problems with duplicate systems

The new service implementation brought many benefits to the organization, as HTTP is a language agnostic protocol, development teams could explore even more technologies and use the right ones for their problems at hand. But when I joined the team, with this scenario at hand, I started to realize that keeping these two implementations alive was introducing too many bottlenecks in the development process, as described below.

####Duplicate effort vs. partial feature availability

For every new feature request to the platform, the development team had to check with the users of “Version3” if they would also need that new feature, and based on that decide if they were implementing it on both systems or not. Bug fixes were a waste as developers had to fix and test them on both systems. We needed to find of getting rid of this duplicate effort.

####Both systems sharing the same database

With both systems sharing that same database, it became more expensive to maintain them, as for every change on any of the systems that triggered a change in the database, that would potentially break the other one, causing a ripple effect through every application using them. As a consequence, the development process became more bureaucratic and less responsive to business demands, as developers needed to double check that both systems were working as expected.

For one of the features, I decided to write down all the times spent on every development stage and create a [value stream map](https://en.wikipedia.org/wiki/Value_stream_mapping) to understand how this delay was impacting our cycle time. I found out that this extra effort in validating systems after a database change took **20% of the whole development process**!

![](/assets/article_images/lessons-learned-from-software-rewrites/vsm.png)

To make things even more exciting, after adding some performance tests to the HTTP implementation, we found out that it had serious performance issues, mostly because it was running on top of a [blocking server architecture](http://stackoverflow.com/questions/8362794/networked-systems-whats-the-difference-between-a-blocking-and-a-non-blocki). Also, some of the core features were tightly coupled with some outdated services that were slowing down significantly response times.

###Our plan to improve this architecture

With that scenario in hand, it was clear that we needed to simplify the overall design to start gaining development speed. Our goal was to have a single implementation, with a straightforward and robust architecture, but we knew we couldn’t have it straight away, so we decided to use an approach inspired on Toyota’s [improvement katas](http://blog.crisp.se/2013/05/14/jimmyjanlen/improvement-theme-simple-and-practical-toyota-kata), to step-by-step head towards that direction.

####Understanding how people are using our services

Before getting our hands dirty we wanted to understand the scope of the work, so we started collecting system usage metrics to find out how our clients were using our services, and from that discover the features that were being used and the others no one cared. The result was going to be used as input for our subsequent efforts, both to eliminate dead features and to prioritize the ones we were going to start implementing the new architecture. We used [Codahale Metrics](http://metrics.dropwizard.io/) for collecting these metrics, but there are plenty of other alternatives out there. After a few weeks, we’ve already had enough data to start electing the first features to be removed.

####Reducing workload by eliminating "dead" features

While working on this project, I met my friend [Joshua Kerievsky](https://www.industriallogic.com/people/joshua) in one of his visits at Globo.com. I was sharing with him some of our learnings and challenges we were facing, then he mentioned a refactoring strategy he teaches in [one of his workshops](https://www.industriallogic.com/onsite-workshops/testing-and-refactoring/), that could be helpful for us to get where we wanted to be. The goal is to remove dead code from your codebase as they introduce unnecessary complexity, in particular on the parts that are integrating with them. I remember once pairing with my colleague [Andre](https://twitter.com/andrerigon); we were tracing down the process of creating a new user, which was slow at the time. To our surprise we found out that's because it was querying the database 31 times within a single execution, mostly doing work related to dead features. Once potential candidates were identified, they were put into a code quarantine for a few weeks to make sure no one was using them, and then deleted completely. We spent a few months doing that and **ended up reducing the number of features by ~40%**, which was a good return on investment.

####Creating a Strangler Application to slowly get rid of the old systems

The metrics we’d collected were also useful to rank the most important features and from that select the ones we would start tackling. For every piece of functionality, we created a suite of [characterization tests](http://c2.com/cgi/wiki?CharacterizationTest) where we verified exactly how it was supposed to behave, and we also took advantage of that to start using a technique called [continuous integration](https://en.wikipedia.org/wiki/Continuous_integration), that allowed us to automatically check, on every build, that it was still working as expected. That increased the level of confidence of the team to implement the functionality on the new stack, which was based on [Scala](http://www.scala-lang.org/) and [Finagle](http://twitter.github.io/finagle/).

We used a technique called [Strangler Application](http://www.martinfowler.com/bliki/StranglerApplication.html) to rewrite every feature into the new stack. As [Martin Fowler](http://martinfowler.com/) describes in his post, *it works by gradually creating a new system around the edges of the old one, letting it grow slowly over several years until the old system is strangled*. The reason why we decided to go with that approach was that the whole process needed to be extremely transparent to the users of our services. To make that happen we’ve implemented a HTTP [Content-Based Router](http://www.enterpriseintegrationpatterns.com/ContentBasedRouter.html) that intercepted every request to our services, and where we could tell which implementation it was going to use to handle specific endpoints. So once a functionality was implemented on the new stack, with all characterization tests passing, all we needed to do was to tell the router to use this new implementation, instead of the old one, and everyone consuming this endpoint would be automatically using the new implementation! This strategy would also allow us to rollback to the old version if something wrong happened.

![](/assets/article_images/lessons-learned-from-software-rewrites/strangle.png)

###Takeaways

Software rewrite is a quite painful process, no matter which strategy you choose to use. On another experience, we decided to rebuild the whole thing in parallel, while the old system was in production, and I remember we dedicating a lot of effort trying to catch up with the old system, as it continued growing. The Strangle Application strategy is the one that worked for me, although it’s a long process to get there. I left the project a year ago and last time I heard from them, they were still working on that. It all will depend on your scenario.
