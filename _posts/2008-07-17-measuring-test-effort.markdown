---
layout:     post
title:      "Measuring test effort"
comments:   true
date:       2008-07-17
permalink:  "measuring-test-effort"
tags:
- agile
- project management
- tdd
- organizational transformation
---
One of the most difficult tasks for consultants is to influence business people to embrace and support test-driven development. Seems like they do "understand" the values, "agree" with that, but when it comes to put into practice the figure is generally a bit different. When I say to put into practice, I mean stick with it steadily, even when dealing with unexpected situations. A typical one could be of a project with delivery delays, a tight deadline, and invariant scope. By experience, when such situation happens, the first decision made is to cut off test development and give way code quality, in order to deliver faster. No matter how hard you try to revert it by showing them the bad outcomes for this decision, they simply ignore them and take the risks, just for the fact that there are no concrete risks, other than not delivering the software.

Not having a way to show managers that not writing tests, at least for the most critical functionalities, is indeed a concrete risk, has always puzzled me. One day while talking to [Kristan Vingrys](http://www.vinktank.com/) about this, he showed me a risk matrix he has been using to help him influencing people to understand test values. See the image:

![](/assets/article_images/2008-07-17-measuring-test-effort/test-matrix.png)

Basically it measures the rate of test coverage required and tells what type of tests (unit, functional) to be implemented based on the impact of the functionality to the business stakeholders and the amount of new code needed to implement it (you can be re-implementing it from an existent code). The more impact and likelihood for new technology the feature needs, the more test implementation it should have.

The ideal approach would be, for each implemented feature, the team is responsible to evaluate and make a decision on how much test effort they want to put in the story. The best time to make it is during the iteration planning meeting, so that the final output you get is both the iteration goal/features and the minimal of test effort to each of them.

And as generally all features has at least a minimum of significant business value, you will always have the guarantee of having these tested.
