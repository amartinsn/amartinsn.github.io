To Be Decided

## How We Are Dealing With Legacy Code


At the end of last year I joined two other developers with the difficult task of improving (replacing would be an option)  and extending the codebase and architecture of a group of systems responsible for authenticating, managing users and their access to all products of my current company. 

At the beginning the systems were comprised of a really old EJB 2.1 API with all business rules, and two web apps— one used by product administrators to define access rules to products, built on Ruby on Rails, and the other used by end users to sign in and up at Globo.com, built using Struts 2.0. 

## Motivations for improving


There were some attempts to improve the existing codebase, some of them.

### Expensive to extend

The main motivation has always been the complexity and the long time needed for testing and extending an EJB architecture. Due to its multilayer architecture, implementing end-to-end tests is really a pain in the ass. In another EJB project I worked on back in 2008, I decided to measure for one day the percentage of the time I spent restarting JBoss servers between on test and another. I ended up giving up after reaching 30%. Every new feature was so expensive!

### Hard to keep developers onboard 

Getting stuck and feeling unproductive

## Getting stakeholders onboard

TBD


## Our approach for improving…


### Ditch unused features


A good start when ….. is to …..

An approach that has been working well for us is finding out which features of the system are actually being used. This was fairly simple and we did it by tracking every HTTP request hitting our web servers and plotting it on a chart. We’ve been tracking it for months, every month electing ones we suspected were not being used, confirmed by our metrics and ditching them after all. Until the day of this write up, we’ve eliminated around 30% of the features, and it’s not done yet. We are using Coda Hale’s metrics and it does the job pretty well.

### Create your safety net


The second step, after finding out features that are actually used on the system is to cover them with a decent suite of tests. I’m not talking about unit testing, because we are still not interested on implementation details. The goal here is look the code harvesting business rules and “documenting” them on what Michael Feathers calls Characterization Tests on his book. Sometimes I can be pretty hard to write those tests, because systems might have lots of outdated rules, that might had made sense at the time they were implemented, but not anymore.

### Kill each feature on the old systems


Instead of following a pattern and creating a new system, running in a cutting edge stack, we decided to pick up the latest one and make it what Martin Fowler calls a strangler application. feature by feature making all the apps pointing out to the old ones to point to the newly implemented feature. We had the guarantee that it was working as we expected, thanks to the characterization tests we implemented as described above. 

### Continuous Delivery FTW.


## Going further….


the main goal is to reduce the number of systems doing the same thing
