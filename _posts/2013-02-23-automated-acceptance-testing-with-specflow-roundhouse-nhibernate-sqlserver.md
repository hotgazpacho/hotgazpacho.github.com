---
layout: post
title: Automated Acceptance Testing with SpecFlow, RoundhousE, NHibernate, & SQL Server
published: true
---

##What? Why?

Over the past couple of years, I've been working on a *very* legacy application. It's a huge, sprawling code base for targeted at a heavily regulated sector. As I've added new features and fixed bugs, I've looked for opportunities to introduce various forms of automated tests to help specify (and verify) the behavior of the application. My team currently has a suite of over 6100 tests, mostly unit tests, but also a few hundred integration tests (mainly because they touch the file system or the database). This is a good thing, but it's not enough; the tests are extremely developer-focused. They don't provide a good high-level understanding of how the parts are supposed to work together. 

We can do better.

###Enter SpecFlow

[SpecFlow](http://http://www.specflow.org/) is tool for practicing [Acceptance Test Driven Development](http://testobsessed.com/2008/12/acceptance-test-driven-development-atdd-an-overview/). It is a .NET port of the Ruby tool [Cucumber](http://cukes.info) (in fact, it uses the same lexer & parser, gherkin, that Cucumber does, thanks to [Ragel](http://www.complang.org/ragel/) and [IKVM](http://www.ikvm.net/)). At its core, SpecFlow is a tool for facilitating communication among the team and developing a shared understanding of how the application is supposed to function. It then provides a way to take those specifications and bind them to the execution of your code. And if we can execute it, we should be able to automate that!

	Feature: Automated Acceptance Tests
		In order to obtain faster feedback on a feature
    	As a developer
    	I want to automate the execution of the acceptance tests

###What about RoundhousE?

As I mentioned, the application that my team works on is a *very* legacy application. It was initially written in a database-first style; data transfer objects were generated from the schema. We've put a halt to this practice, but it was in use for almost 7 years before that. Suffice it to say that, besides the legacy code, we are left with a substantial legacy database. One of the first things I did when I joined the team was to get the schema under source control. 

Having dabbled in Rails, I loved the idea of having a set of database migrations that evolved the schema of the database over time. I chose to use [RoundhousE](https://github.com/chucknorris/roundhouse/wiki), because it uses native SQL to express the migrations. I also liked that it could create a new database by first restoring a SQL Server backup, then playing the migrations over that. This allowed me to draw a line in the sand; from the time of the backup I took forward, all changes to the database schema would be expressed as migration scripts and stored with the source code to the application.

This has worked out *extremely* well for us. No more having to use the Visual Studio schema compare tools to keep the multitude of client databases in synch. It has also allowed us to include automated tests for the persistence of our entities (via NHibernate), so we know fairly quickly if a code change to an entity, or a database schema change, will cause the application to fail.

###Back to SpecFlow

Alright, so we're automating our acceptance tests. To have a reasonable level of confidence, we need to test with as complete a stack as is feasible. We're going to skip testing against the UI. From *[7 Deadly Sins of Automated Software Testing](http://www.agileengineeringdesign.com/2012/01/7-deadly-sins-of-automated-software-testing/)* by Dr. Adrian Smith:
> Although automated UI tests provide a high level of confidence, they are expensive to build, slow to execute and fragile to maintain.... UI based tests should only be used when the UI is actually being tested or there is no practical alternative.

In other words:

![It's a trap!](http://laughingsquid.com/wp-content/uploads/its-a-trap-20100127-143341.jpg "It's a trap!")

That's all well and good, but doesn't the [Cucumber wiki](https://github.com/cucumber/cucumber/wiki/Given-When-Then) direct us to **observe outcomes**? It does, in fact:

>The observations should be related to the business value/benefit in your feature description. The observations should also be on some kind of output – that is something that comes out of the system (report, user interface, message) and not something that is deeply buried inside it (that has no business value).

How will we accomplish this without testing the UI? Well, for new features, we're building screens using the [Supervising Controller pattern](http://martinfowler.com/eaaDev/SupervisingPresenter.html). All user gestures are forwarded *(via .NET events that speak in terms of the domain, NOT the UI)* to a controller, which processes the input and directs the view *(the controller holds a reference to the view, via an Interface)* to change in response. That is, we follow the ["Tell, don't ask"](http://pragprog.com/articles/tell-dont-ask) principle. Therefor, we can mock the view's interface, provide that to the controller, and verify in our [step definitions](https://github.com/techtalk/SpecFlow/wiki/Step-Definitions) that the view received the correct messages.

What about the database, though? Well, we'd like to test as much of the stack, from top to bottom, as possible. Seems reasonable that we'd want to include the lowest level of the stack, the database, in our acceptance tests. However, databases are the ultimate in global shared state. How can we have any confidence that the outcomes **or side effects** of previously executed scenarios won't influence the outcome of subsequent scenarios? 

The most certain way is to provide a clean database for each scenario. Running each scenario in a transaction is out, as our current architecture allows, nay encourages, Command objects, described by [Ayende's blog series on Limiting Abstractions](http://ayende.com/blog/154209/limit-your-abstractions-refactoring-toward-reduced-abstractions), to manage their own transactions. We could probably refactor out a Command Executor object that manages the transactions for the commands, and remove transactions from all commands. That way, we could inject an executor for tests that no-ops on commit, but have runtime one that does a real commit. However, that's a daunting task due to the sheer number of commands in the system. I'm also not certain that I see the value in it.

###The Hook Brings you Back

This leaves us with having a fresh database for each scenario. Fortunately, SpecFlow [provides hooks](https://github.com/techtalk/SpecFlow/wiki/Hooks) for interesting events in the test lifecycle. One such pair is BeforeScenario/AfterScenario. With this, we can create a new database for the scenario before it runs. RoundhousE makes this easy. We can restore our backup to a new database and run the scripts on that. One problem: the restore & migrate process take about 33 seconds(!) on my dev machine (Core i7 Quad with an OWC Mercury Extreme Pro 6G SSD). Multiple minutes *per feature* spent just on *creating the database* is simply not acceptable.

We can do better.


<iframe width="420" height="315" src="https://www.youtube-nocookie.com/embed/pdz5kCaCRFM" frameborder="0" allowfullscreen="allowfullscreen">Blues Traveler's The Hook</iframe>

Can we pay the restore & migrate penalty just once? **Sure can!**

How can we do that while providing a clean database for each scenario?

We use SQL Server. SQL Server provides the ability to [Detach and Attach the data files](http://msdn.microsoft.com/en-us/library/ms190794.aspx) for a database. We could restore & migrate an example database once, detach it, copy the MDF files to another location, and reattach the example in a BeforeTestRun hook. Then, we can use the BeforeScenario hook and information contained in the [ScenarioContext](https://github.com/techtalk/SpecFlow/wiki/ScenarioContext) to copy the template MDF to a new file and attach that as a new database specifically for the scenario. This is **orders of magnitude faster** than the restore & migrate method. We can then clean up the databases in the AfterScenario hooks, and clean up everything in the AfterTestRun hook.

And what about NHibernate? Creating the SessionFactory is an expensive (though not nearly as expensive as the database itself) operation. We only want to do this once, but we need to be able to switch the connection string we use for each scenario. No problem! NHibernate provides a ton of hooks into the system, and [this article on NHForge](http://nhforge.org/wikis/howtonh/dynamically-change-user-info-in-connection-string.aspx) describes the approach to varying the connection string.  

Put it all together, and here's the code:

<script src="https://gist.github.com/hotgazpacho/5022723.js"><!--TestEnvironment.cs--></script>
