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

![](/assets/article_images/lessons-learned-from-software-rewrites/initial.png)

### Consequences of rewriting from scratch

The new API was indeed much easier to use than the other one and was fully tested, which gave the team much more confidence to extend it. But writing a new system from scratch also introduced some undesired consequences.

With the new implementation ready to rollout, clients needed to prioritize this migration on their backlogs. Most of them did it but a few ones didn't, preventing the team from shutting the old system down. That was the scenario I found when I joined them in 2013.

![](/assets/article_images/lessons-learned-from-software-rewrites/joining.png)

As you can see from the picture above, there are two APIs doing pretty much the same thing, one used by most of our clients and another still used by a few ones. And to make things worse, both systems share the same database, which today is our most critical pain point.

There's a lot out there talking about the problems of having shared databases and I won't go into details on that. Sam Newman wrote about that on his new book Building Microservices. Also Martin Fowler describes this pattern (or anti-pattern) in greater detail on one of his signature series books, Enterprise Integration Patterns.

For us, the main problem was that, for every changes on either systems, such as new features or bug fixes, which triggered a change in the database, it would potentially break the other system, causing a ripple effect through every application that uses it. As a result, any database change required an extra effort from the team maintaining the other system, which was turning our development process much less responsive to business demands. This is what Mary and Tom Poppendieck calls demand of failure. Definitely not a wise way to spend your company's money.

### Our plan to improve this architecture

With the scenario described at hand, our main goal became to seamlessly migrate the rest of our clients to the new API. But we didn't want to depend on them to be able to do that, so we decided to turn the new API into a Strangler Application. By doing that, we could transition users feature-by-feature incrementally, in constant releases. Once no one was using a feature we could then delete it from the codebase. We knew the effort was going to be humongous, so we needed to somehow find a way to make this process more effective.

#### Reducing workload by looking for "dead" features

A while ago I had a chat with my friend Joshua Kerievsky during one of his visits at Globo.com. We were talking about refactoring strategies and he mentioned one of the strategies he teaches on his refactoring workshop. The idea is to look for dead code on your codebase, those no one was using anymore. By doing that you can also end up finding dead features. Once you find those you can put them on a sort of code quarantine during a period of time, and then delete them completely from your codebase.

There's also a study presented by Jim Johnson, chairman of the Standish Group, on XP 2002, which states that 45% of features on a software system are never used, and only 20% are used often or always.

Both points were quite useful and inspired us on our first action to turn this process more effective. We decided to start collecting API usage metrics, so we could identify our dead features. After a period of time observing we started categorizing features by the following criteria: those we were sure no one was using and those we were not sure. For those we were sure we started deleting them straight away, and for the other group, we left them for a long period of time and we did some investigation with clients. Once confirmed no one was using, we also deleted them.

This strategy reduced the amount of features we'd have to strangle by ~40%, which was a good investment.  

#### Create your safety net

The strategy we wanted to use was to implement some kind of smart HTTP proxy (http://www.enterpriseintegrationpatterns.com/ContentBasedRouter.html), and make it intercept every call to our APIs

There’s also a principle called Pareto principle, also known as the 80-20 rule, which states that, for many events, roughly 80% of the effects come from 20% of the causes. This is perfectly applicable to software development
Rewrite feature by feature, starting with the most critical ones
Build your safety net
highest risk features….
