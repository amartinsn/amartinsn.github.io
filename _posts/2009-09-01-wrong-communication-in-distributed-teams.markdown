---
layout:     post
title:      "Wrong Communication In Distributed Teams"
comments:   true
date:       2009-09-01
permalink:  "wrong-communication-in-distributed-teams"
tags:
- agile
- project management
- organizational transformation
---
One of the big challenges faced by distributed teams is how to get over the communication gap created by the physical distances that separates them. We all know that communication, either verbal or non-verbal, is fundamental for any project to be delivered successfully. When a team is good at communicating, they cultivate a more effective sense of collectivity and cooperation, having faster feedback, by sharing information (knowledge) and having valuable discussions.

But this is not quite the real world for distributed development teams. It’s much harder, not to say almost impossible, to know what exactly is happening on each other’s mind. What problems and technical challenges are they facing? What are they doing now? What points are they considering when designing a new feature? How important is for them to write tests? Are they following the project development standards?

### Blame the "Bandwidth Limited" Communication Tools!

> Software development teams, by the nature of their work, needs to discuss and assess different ideas to solve complex problems. And they are very difficult to communicate when using tools such as email or telephone, which on the book they call “bandwidth limited”. And those are exactly the ones available for most distributed teams. So face-to-face communication suits better for this kind of discussions, using the assistance of diagrams or sketches, not to mention the use of body language. This would give us immediate feedback, just by looking into the other person’s eyes, which communicate understanding.

<cite>[The Organization and Architecture of Innovation: Managing the Flow of Technology](http://tinyurl.com/kpoxmx) (with some modifications)</cite>

And as you can't always minimize distances to allow verbal communication, you have to look for other ways, and maximizing non-verbal communications is definitely a road to go down.

### Some Bad Outcomes

#### Poor Code Quality

- **Code Duplication** ([see](http://www.c2.com/cgi/wiki?DuplicatedCode))
It's quite usual. For example, the guy wants to load a XML file as a String so that he can perform some assertions over the result. He will implements something like a
FileLoader class. But what he doesn't know is that another developer has already implemented a class with this behaviour.

- **Reinventing the wheel** ([see](http://en.wikipedia.org/wiki/Reinventing_the_wheel))
This is partially caused by lack of communication and partially a result of the programmer's discipline. When adding a new library to the project the team must have a discussion and look for the benefits earned by using it. Before adding a XML parsing library that you're used to, have a quick chat with the team will let you know if is there any other parsing library being used. Maybe someone could make a walk-through with you on it. But it is your responsibility to know how to use it afterwards.

- **Code For The Others** (and for yourself)
When coding, you should always ask yourself if your peers would be able to understand what are you producing. Better still, you should ask if you would easily understand it again in a couple of weeks from now. It's quite common when coding, you get contextualized with what you need to do to deliver that functionality. This context will always get lost after finishing, unless you share it with the others or document it. There are [some](http://tinyurl.com/m73tjb) [good](http://tinyurl.com/3jms4t) [materials](http://tinyurl.com/lxr9ke) [out](http://tinyurl.com/n34a2h) [there](http://tinyurl.com/mkh47l) that [shows](http://tinyurl.com/lxj2qn) you how to write clean and readable code.

- **Broken Builds**
In a distributed team, a broken build not only just affects the people in your room, it also affects people in rooms into other cities. So reverting a broken build should be taken into account, specially when you have a slow build, then definitely the commiter would get himself into a big problem! Imagine a long build that takes about 30 minutes for example, and someone commits something broken. If he fixes it really quickly, it still may take 1 hour for the other team to be able to commit its changes and consequently 1 hour and a half lost in productivity in the other cities. It's all about communication - the quicker the build, the quicker the feedback. So a fast and successful build is mandatory!

#### Fear of Refactoring

Poor code quality results in fear of refactoring. Who hasn't been in a situation, working on a tightly coupled system, where it was quite hard to do any refactoring? Any attempt would propagate the changes deep in the source code, ending up [shaving the yak](http://www.catb.org/~esr/jargon/html/Y/yak-shaving.html), not going anywhere.

#### Absence of Trust

I see this one as a result of the other two I mentioned above. When your team is biased to go off the tracks when trying to comply with code standards, some precautionary measures are generally created to avoid the worse.

I've seen a case where a pair, assigned to implement a story, and almost completing the development, ended up realising that another pair was also looking at it. Don't ask me why!

I've also seen people creating triggers on the version control system, so that for each commit from one team, the other received an email with all the commit information. This is good in one side, because you can easily identify cowboy commiters that don't write tests. But this is also used to check if the code is acceptable, reverting if not!

### My Current Experience

The team I'm currently working with is facing some of these problems, and during all last week, when I was on the other side of the fence, visiting the other part of out team on Tasmania, this became even more highlighted. Although we were having daily stand-up meetings, I felt like I was missing something, specially because there was another team in Melbourne joining us and I still didn't know how it was going to work out. Chatting with my friend [Mark Needham](http://www.markhneedham.com/) about it, he recommended me a book called [The Organization and Architecture of Innovation: Managing the Flow of Technology](http://tinyurl.com/kpoxmx), where there's a chapter dedicated exclusively to this point, and that I could probably get some ideas of how to overcome this problem.

#### Taking actions

> It seems obvious that an organization that wants its technical staff members to communicate needs to ensure the distances among them are minimized. Unfortunately, the traditional and most common form of office configuration does just the opposite. Not to mention when they are in separated buildings.

The quote above also extracted from the book, doesn't tell anything new, and that's exactly one of the issues we wanted to fix. Now, with three teams we agreed that we would need to have them communicating face-to-face more often. So the rule is that every week we should have at least one person from each team visiting a different one. Apart from that, we are continuing with our [daily stand-up meetings](http://en.wikipedia.org/wiki/Stand-up_meeting), each team separately, and later on another daily meeting, but between teams (in the Scrum world called [Scrum of Scrums](http://agilecommons.org/posts/d551a84f06)). This one involves, by default, only the iteration manager and the tech lead, but everyone else is also welcome to attend.

We also had to put more effort on improving the non-verbal communication, as they are more required on distributed teams. With this separation teams have to be even more strict with what they permit or not in the codebase. We introduced development tools such as [Checkstyle](http://checkstyle.sourceforge.net/) and [compile with walls](http://tinyurl.com/ktn6mn) to ensure this. Checkstyle acts as a hammer on misbehaved commiters and Compile With Walls ensures that project structure is being respected. Sometimes quite good threads (over IM or email) are created by people trying to understand why a Checkstyle rule has failed.
