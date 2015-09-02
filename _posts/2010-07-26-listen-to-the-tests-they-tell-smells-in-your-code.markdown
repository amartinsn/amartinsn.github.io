---
layout:     post
title:      "Listen to the tests, they tell smells in your code..."
date:       2010-07-26
comments:   true
tags:
- ruby
- tdd
- object-oriented design
---

Recently, reading [Growing Object Oriented Software Guided Tests](http://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627), written by [Steve Freeman](http://www.m3p.co.uk/blog) and [Nat Pryce](http://www.natpryce.com), it reminded me of a project I worked on a while ago. It was a one year old system, poorly tested, integrating to a handful of other systems, and the code-base... well I prefer not to remember. Despite this scenario, I joined the team to help them implement some new functionalities.

I remember sometimes it was difficult to write tests, the classes were tightly coupled, with no clear responsibilities, several attributes, bloated constructors, etc. And despite our best effort, working around the bits that were preventing us from writing the tests, we felt we were getting down the wrong road, trying to do it in such a crappy code-base. As a result some of our tests were massive! A bunch of lines of mocks, stubs, and expectations, making it impossible to understand their purpose.

### What have I learned?

Reading one of the book chapters I learned that the same qualities that makes an object easy to test also makes the code responsive to change. In my situation, the tests were telling me how clumsy the code was and how difficult it would be to extend it.

I also learned that when we come across a functionality that is difficult to test, asking ourselves how to test it is not enough, we also have to ask why is it difficult to test, and check whether it's an opportunity to improve our code. The trick is to do it driven by tests, so we can get rapid feedback on code's internal qualities and on whether it's doing what it's supposed to do.

So they introduced a variation for the well-known TDD cycle— _**"Write a failing test" => "Make the test pass" => "Refactor"**_. As described on the figure below (extracted from the book), if we're finding it hard to write the next failing test for our application, we should look again at the design of the production code and often refactor it before moving on, until we get to the point that we can write tests that reads well.

![Extracted from Growing Object-Oriented Software, Guided By Tests— Steve Freeman and Nat Pryce](/assets/article_images/2010-07-26-listen-to-the-tests-they-tell-smells-in-your-code/tdd-cycle.png)

Extracted from Growing Object-Oriented Software, Guided By Tests— Steve Freeman and Nat Pryce

###An Example of a Smell Tests Might Be Telling You

####Reference data rather than behavior

When applying ["Tell Don't Ask"](http://pragprog.com/articles/tell-dont-ask) or ["Law of Demeter"](http://pragprog.com/articles/tell-dont-ask) consistently, we end up with a coding style where we tend to pass behavior into the system instead of pulling values up through the stack. So picking up the famous Paperboy example, before refactoring the code applying the "Law of Demeter" the code and test would look something along the lines of the snippet showed below.

```ruby
class Paperboy
  def collect_money(customer, due_amount)
    if due_amount > customer.wallet.cash
      raise InsuficientFundsError
    else
      customer.wallet.cash -= due_amount
      @total_collected += due_amount
    end
  end
end
```
```ruby
it "should collect money from customer" do
  customer = Customer.new :wallet => Wallet.new(:amount => 200)
  paperboy = Paperboy.new
  paperboy.total_collected.should == 0
  paperboy.collect_money(customer, 50)
  customer.wallet.cash.should == 150
  paperboy.total_collected.should == 50
end
```

We can easily see that the test is telling us it knows too much detail about Customer class implementation. We can see its internals, which objects it's related to, and even worse, we're also exposing implementation details of its peers. So it's clear for me that it needs some design improvement. My main goal here is to hide Customer implementation details from the users of the Paperboy class. Which means that I don't wanna see anything but Customer and Moneyclasses referenced on the test!

```ruby
class Paperboy
  def collect_money(customer, due_amount)
    @collected_amount += customer.pay(due_amount)
  end
end
```
```ruby
it "should collect money from customer" do
  customer = Customer.new :total_cash => 200
  paperboy = Paperboy.new
  paperboy.total_collected.should == 0
  paperboy.collect_money(customer, 50)
  customer.total_cash.should == 150
  paperboy.total_collected.should == 50
end
```

The method ```customer.pay(due_amount)``` wraps all the implementation detail up behind a single call. The client of paperboy no longer needs to know anything about the types in the chain. We've reduced the risk that a design change might cause ripples in remote parts of the codebase.

As well as hiding information, there's a more subtle benefit from "Tell, Don't Ask." It forces us to make explicit and so name the interactions between objects, rather than leaving them implicit in the chain of getters. The shorter version above is much clearer about **what** it's for, not just **how** it happens to be implemented.

All the logic necessary to collect the money is inside the ```Customer``` object, so it doesn't have to expose its state to its peers.

Now it's safer to continue writing new failing tests to our objects. Remember, listen to the tests!
