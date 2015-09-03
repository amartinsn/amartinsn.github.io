---
layout:     post
title:      "Lessons Learned From Software Rewrites"
comments:   true
tags:
- architecture
- software design
- refactoring
---

By the end of 2013 I joined a team challenging myself to help them improve one of the most critical products on my company’s portfolio. One of its components was an API, built back in 2004 and was used by pretty much all other products in our organization. This API was implemented using Java and the EJB stack. It had two interfaces with clients, the EJB client interface and a feature incomplete HTTP (RESTfulish) interface, built on top of that EJB client.

A couple of years before I join this team, a group of developers made a great effort to rewrite this API from scratch. They continued using Java but chose other technologies such as Spring and Jersey, which was a good decision in my opinion. The features were now exposed as RESTful webservices and most importantly, they introduced a decent suite of tests, which was quite hard on the EJB stack. The new architecture is better described on the picture below:

***** picture of the architecture* ______


### Consequences of rewriting from scratch

The new API was indeed much easier to use than the other one and was fully tested, which gave the team much more confidence to extend it. But writing a new system from scratch also introduced some undesired consequences.

With the new implementation ready to rollout, clients needed to prioritize this migration on their backlogs. Most of them did it but a few ones didn't, preventing the team from shutting the old system down. And that was the scenario I found when I joined them in 2013.

***** picture of the architecture after the rewrite* ______

As you can see from the picture above, there are two APIs doing pretty much the same thing, one used by most of our clients and another still used by a few ones. And to make things worse, both systems share the same database, which today is our most critical pain point.

There's a lot out there talking about the problems of having shared databases and I won't go into details on that. Sam Newman wrote about that on his new book Building Microservices. Also Martin Fowler describes this pattern (or anti-pattern) in greater detail on one of his series books, Enterprise Integration Patterns.

For us, the main problem was that, for any changes on either systems, such as new features or bug fixes, which triggered a change in the database, it would potentially break the other system, causing a ripple effect through every application that uses it. As a result, any database change required an extra effort from the team maintaining the other system, which was turning our development work much less responsive to business demands.

This is what Mary and Tom Poppendieck call a demand of failure, not a wise way to spend your company's money.

### Our plan to fix it

The scenario I described above was the one I found when I joined this team, and with the takeaways from my previous experience I started looking how people were dealing with software rewrites out there. The goal was to seamlessly move everyone still using the old system to the new one, so we could kill it! (explore more shared database problem?).

What we needed was to create what Martin Fowler calls Strangler Application, and migrate incrementally every feature to it and then deleting from the old codebase. Our new API was a perfect fit to be a Strangler Application. Halfway done with it! Now we just needed to make all our users use it whilst deleting the old implementations (this is too repetitive). The strategy we wanted to use was to implement some kind of smart HTTP proxy (http://www.enterpriseintegrationpatterns.com/ContentBasedRouter.html), and make it intercept every call to our APIs

First thing that came to my mind was that the effort was going to be humongous, so we needed somehow find a way to make it more effective.

I came across a post Martin Fowler wrote ages ago talking about the XP Conference. On that post Martin mentions a talk given by Jim Johnson, chairman of the Standish Group. Jim presented a study conducted by his company, which states that 45% of features on a software system are never used, and only 20% are used often or always. That post made me remind of a chat I had with my friend Joshua Kerievsky during one of his visits at Globo.com. We were talking about refactoring strategies and he said I should always look for dead code and dead features-- those that are hanging there but no one is actually using them--, and ditch them out of the codebase.

That was a really good starting point, if we were going to invest our money doing a rewrite,
From that we start collecting usage metrics

There’s also a principle called Pareto principle, also known as the 80-20 rule, which states that, for many events, roughly 80% of the effects come from 20% of the causes. This is perfectly applicable to software development
Rewrite feature by feature, starting with the most critical ones
Build your safety net
highest risk features….

### Strangler Application ###
