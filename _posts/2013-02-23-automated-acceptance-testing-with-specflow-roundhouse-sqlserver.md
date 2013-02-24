---
layout: post
published: false
---

#Automated Acceptance Testing with SpecFlow, RoundhousE, & SQL Server

##What? Why?

Over the past couple years, I've been working on a *very* legacy application. It's a huge, sprawling codebase for targetted at a heavily regulated sector. As I've added new features and fixed bugs, I've looked for oportunities to introduce various forms of automated tests to help specify (and verify) the behavior of the application. My team currently has a suite of over 6100 tests, mostly unit tests, but also a few hundred integration tests (mainly because they touch the file system or the database). This is a good thing, but it's not enough; the tests are extremely developer-focused. They don't provide a good high-level understanding of how the parts are supposed to work together. 

We can do better.

###Enter SpecFlow

[SpecFlow](http://http://www.specflow.org/) is tool for practicing [Acceptance Test Driven Development](http://testobsessed.com/2008/12/acceptance-test-driven-development-atdd-an-overview/). It is a .NET port of the Ruby tool [Cucumber](http://cukes.info) (in fact, it uses the same lexer & parser, gherkin, that Cucumber does, thanks to [Ragel](http://www.complang.org/ragel/) and [IKVM](http://www.ikvm.net/)). At it's core, SpecFlow is a tool for facilitating communication among the team and developing a shared understanding of how the application is supposed to function. It then provides a way to take those specifications and bind them to the execution of your code. And if we can execute it, we should be able to automate that!

	Feature: Automated Acceptance Tests
		In order to obtain faster feedback on a feature
    	As a developer
    	I want to automate the execution of the acceptance tests

As I mentioned, the application that my team works on is a *very* legacy application. It was initially authored in a database-first style; data transfer objects were generated from the schema. We've put a halt to this practice, but it was in use for almost 7 years before that. Suffice it to say that, in addition to the legacy code, we are left with a substantial legacy database. One of the first things I did when I joined the team was to get the schema under source control. 

Having dabbled in Rails, I loved the idea of having a set of database migrations that evolved the schema of the database over time. I chose to use [RoundhousE](https://github.com/chucknorris/roundhouse/wiki), because it uses native SQL to express the migrations. I also liked that it could create a new database by first restoring a SQL Server backup, then playing the migrations over that. This allowed me to draw a line in the sand; from the time of the backup I took forward, all changes to the database schema would be expressed as migration scripts and stored with the source code to the application.
