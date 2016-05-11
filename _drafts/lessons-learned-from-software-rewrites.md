---
layout:     post
title:      "Lessons Learned From Software Rewrites"
comments:   true
tags:
- architecture
- software design
- refactoring
---

By the end of 2013 I joined a team challenging myself to help them improve their software ecosystem, which is part of one of the most critical areas inside the organization. The most important component in this ecosystem is the Java + EJB API, built back in 2004, responsible for registering, authenticating, and granting users access to all products inside the organization. It had two main interfaces to the outside world, the Java client and a feature incomplete HTTP webservice built on top of the Java client, as shown on the diagram:

![](/assets/article_images/lessons-learned-from-software-rewrites/initial.png)

A couple of years before I join this team, a group of developers made a great effort to rewrite this API from scratch. All features were mapped, reimplemented on a new stack, and exposed as RESTful webservices. The team also took advantage of this effort and introduced a decent suite of automated tests, which was really painful to accomplish in the EJB stack. The new architecture is  described on the picture below:

![](/assets/article_images/lessons-learned-from-software-rewrites/rebuild.png)

### Consequences of rewriting from scratch

The new API was indeed easier to use, and the introduction of automated tests gave the team much more confidence to continue improving it. But it also introduced some undesired consequences.

#### Duplicate implementations

With the new API released most of the users migrated to the new implementation, but for some other ones, the cost of performing this migration was so expensive that product managers ended up never prioritizing it, as they would have to compromise features that were much more valuable to them. This scenario blocked the team from shutting down the old system, and since then we've been carrying the cost of having two services for the same purpose.

#### The problem with shared databases

There's a lot out there talking about the problems of having shared databases between systems. For us, the main problem was that, for every change on any of the systems that triggered a change in the database, that would potentially break the other systems, causing a ripple effect through every application using them.

As a result, our development process ended up tailored to try to mitigate this risk. Every database change required an extra effort from the teams maintaining the other systems to make sure nothing was going to break. That turned our development process much less responsive to business demands.

### Our plan to improve this architecture

With the scenario described at hand, our main goal became to seamlessly migrate the rest of our clients to the new API. But we didn't want to depend on them to be able to accomplish that, so we decided to turn the new API into a Strangler Application. By doing that we could transition users, feature-by-feature, to the new implementation and then delete the old implementation from the codebase.

Our strategy was to implement some kind of HTTP [Content-Based Router](http://www.enterpriseintegrationpatterns.com/ContentBasedRouter.html) to intercept every call to both APIs, where we could specify which API implementation would handle a request to given endpoint. Once we decide that a feature **A** is now handled by the new implementation, all we needed to do was configure that on the router, and implement a [filter](http://www.enterpriseintegrationpatterns.com/patterns/messaging/PipesAndFilters.html) to translate the response to the old contract, expected by the client. It's essential to be able to switch between implementations, in case of a need to rollback the transition.

![](/assets/article_images/lessons-learned-from-software-rewrites/strangle.png)

We knew the effort was going to be humongous, so we needed to somehow find a way to make this process more effective.

#### Reducing workload by looking for "dead" features

A while ago I had a chat with my friend Joshua Kerievsky during one of his visits at Globo.com. We were talking about refactoring strategies and he mentioned one of the strategies he teaches on his refactoring workshop. The idea is to look for dead code on your codebase, those no one was using anymore. By doing that you can also end up finding dead features. Once you find those you can put them on a sort of code quarantine during a period of time, and then delete them completely from your codebase.

There's also a study presented by Jim Johnson, chairman of the Standish Group, on XP 2002, which states that 45% of features on a software system are never used, and only 20% are used often or always.

Both points were quite useful and inspired us on our first action to turn this process more effective. We decided to start collecting API usage metrics, so we could identify our dead features. After a period of time observing we started categorizing features by the following criteria: those we were sure no one was using and those we were not sure. For those we were sure we started deleting them straight away, and for the other group, we left them for a long period of time and we did some investigation with clients. Once confirmed no one was using, we also deleted them.

This strategy reduced the amount of features we'd have to strangle by ~40%, which was a good investment.  

#### Creating our safety net



There’s also a principle called Pareto principle, also known as the 80-20 rule, which states that, for many events, roughly 80% of the effects come from 20% of the causes. This is perfectly applicable to software development
Rewrite feature by feature, starting with the most critical ones
Build your safety net
highest risk features….
