---
layout:     post
title:      "Lessons Learned From Software Rewrites"
comments:   true
tags:
- jekyll
- code
- fulano
- adulto
- design
- domain-driven design
- object-oriented programming
- architecture
- software
- lessons learned
- globo.com
- qualquer
- coisa
- que
- faça
- encher
- aquela
- lista
- de
- tags
- inúteis
summary:    "This is a post summary I wanna see on the tags page."
---

## Throw It Away and Build From Scratch? ##

First time I joined a project with a demand to improve an existing system, we decided to rebuild it all again from scratch. For us, that was a common sense, as none of us liked the stack and the code was really difficult to understand. So starting from scratch seemed to be the easiest way to go. Green field project, cutting edge stack, perfect! Well, not really...

As the old system was still being used, requests for bug fixes, improvements and new features continued to pop up frequently, and ensuring we were caught up with all that was really time-consuming. Not to mention we ended up spending a lot of time re-implementing features no one was using, which was pointless.

Another problem is the transition to the new system once it’s ready to be integrated. It might take a while, as other teams need to prioritize this work on their backlogs, and in the meantime, the other system might continue changing and there you are, left behind again.

So one lesson I learned from this experience was that, unless you have a really good reason, don’t rewrite everything from scratch. There is a much better way I will describe later on.

## Applying Some Lessons Learned ##

By the end of 2013 I joined a team challenging myself to help them improve one of the most problematic and important products on my company’s portfolio. The most critical component of this product is the API, built back in 2004 using Java and EJB. It had two interfaces with users, the EJB client interface and a feature incomplete HTTP (RESTfulish) interface, built on top of that EJB client.

A group of developers made a great effort to rewrite the whole API from scratch in 2011. They rebuilt it on top of a more recent stack, exposing features as RESTful webservices and most importantly, they introduced a decent suite of tests, which was quite hard on the EJB stack. This new architecture is better described on the picture below:

***** picture of the architecture* ______


### Problems with the new architecture ###

The new API was indeed much easier to use than the other one, but it also brought up some really bad consequences.

Some clients didn't prioritize the effort to migrate to the new API, preventing the team from turn the old implementation off. As a result, it ended up introducing a really bad problem, two systems sharing the same database schema.

There's a lot out there talking about the problems of having a shared database schema. Sam Newman recently wrote about that on his new book. Also Martin Fowler describes this pattern (or anti-pattern) in greater detail on a book called Enterprise Integration Patterns. And from this book I extracted a snippet describing the exact problem it was causing in our company.

> The fact that Shared Database has unencapsulated data also makes it more difficult to maintain a family of integrated applications. Many changes in any application can trigger a change in the database, and database changes have a considerable ripple effect through every application. As a result, organizations that use Shared Database are often very reluctant to change the database, which means that the application development work is much less responsive to the changing needs of the business. *- Enterprise Integration Patterns*

### Our plan to fix it ###

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
