---
layout:     post
title:      "Acceptance Tests With JBehave, Selenium & Page Objects"
comments:   true
date:       2008-11-14
permalink:  "acceptance-tests-with-jbehave-selenium-page-objects"
tags:
- tdd
- java
- bdd
- page objects
---
Since [JBehave 2.0](http://jbehave.org) was released in September, I've been using it on my current project to verify the acceptance criteria for the features we are implementing, ensuring that the web interface is following the right workflow, and is displaying the data as expected, as well as some other important elements.

### What is JBehave?
JBehave is a framework for [Behaviour-Driven Development](http://behaviour-driven.org), that allows customers and developers to work together more closely on features for the system. They get together to discuss and define a set of executable criteria (scenarios) for each of the features that will be used to determine if they are fully implemented or not. The cool thing is that the scenario execution produces an easy-to-read output, so that customers can keep track of the implementation status.

### An Example
In this example, I'm not covering all the bits and pieces about writing and running acceptance tests with JBehave. If you want to see it in details, there's a very good introduction article [here](http://www.ryangreenhall.com/articles/bdd-by-example.html). The intention is to show how we are doing it on my project, integrating with [Selenium](http://selenium.openqa.org/) and using the [Page Objects](http://code.google.com/p/webdriver/wiki/PageObjects) pattern.

#### New Feature

Suppose the customer wanted to provide exclusive content for the user of the website. For that, we would have to implement an authentication framework, so that we could guarantee that only registered users would access that content. So the story looks like this:

```
As an user
I want to login into the website
So that I can view the exclusive contents
```

#### Defining Scenarios

```
Scenario: Invalid Username
Given the user is on the login page
 When the user types username wrong!
  And the user types password 123456
  And clicks the login button
 Then the page should display Invalid Authentication message

Scenario: Successful Login
Given the user is on the login page
 When the user types username alexandre
  And the user types password 123456
  And clicks the login button
 Then the user should be redirected to home page
  And the page should display welcome message to alexandre
```

Each scenario needs its own implementation, providing the necessary steps to be verified, in our case the LoginSteps class. Another improvement on version 2.0 is that it allows you to define multiple scenarios in a single file.

```java
public class LoginScenarios extends Scenario {
	public LoginScenarios() {
		super(new LoginSteps());
	}
}
```

#### Defining Steps (Integrating Selenium)

After defining the scenarios on the text file, it's time to map each scenario step to its implementation. Once you start mapping them you will come across ones that are already mapped, and will realise that defining new scenarios is just a matter of combining existing steps.

Initially, for each step, we were extracting the code sniped from Selenium IDE record and placing into it. But it felt a bit clumsy, with code duplication in some spots. It quickly reminded me how hard it is to maintain a suite of tests when it starts evolve and there is not enough effort on refactoring. We wanted to avoid duplication and come up with a more elegant and reusable solution.

One day my friend Uday Rayala came up with the idea of using the Page Objects pattern for writing our acceptance tests, so that we could encapsulate the logic to interact and verify page state into these objects. The first time I heard about it was from [Simon Stewart](http://pubbitch.org/blog/), when he was in Sydney for the JAOO conference. He said that they implemented [WebDriver](http://code.google.com/p/webdriver/) based on this pattern, and showed me some examples. Here is a definition, extracted from the WebDriver wikipage:


> Within your web app's UI there are areas that your tests interact with. A Page Object simply models these as objects within the test code. This reduces the amount of duplicated code and means that if the UI changes, the fix need only be applied in one place.

```java
public class LoginSteps extends Steps {

	@Given("the user is on the login page")
	public void theUserIsOnTheLoginPage() {
		LoginPage loginPage = new LoginPage();
		loginPage.verifyPresenceOfUsernameAndPasswordFields();
		loginPage.verifyPresenceOfLoginButton();
	}

	@When("the user types username $username")
	public void theUserTypesUsername(String username) {
		loginPage().typeUsername(username);
	}

	@When("the user types password $password")
	public void theUserTypesPassword(String password) {
		loginPage().typePassword(password);
	}

	@When("clicks the login button")
	public void clicksTheLoginButton() {
		loginPage().login();
	}

	@Then("the page should display $errorMessage message")
	public void thePageShouldDisplayErrorMessage(String errorMessage) {
		loginPage().verifyPresenceOfErrorMessage(errorMessage);
	}

	@Then("the user should be redirected to home page")
	public void theUserShouldBeRedirectedToHomePage() {
		HomePage homePage = new HomePage();
		homePage.verifyPage();
	}

	@Then("the page should display welcome message to $user")
	public void thePageShouldDisplayWelcomeMessage(String user) {
		homePage().verifyPresenceOfWelcomeMessageTo(user);
	}

	private LoginPage loginPage() {
		return (LoginPage) Page.current();
	}

	private HomePage homePage() {
		return (HomePage) Page.current();
	}
}
```
