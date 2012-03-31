---
layout: post
title: "Testing with NHibernate in-memory using SQLite"
---
## Background

At work, we've settled on an MVP, passive view pattern for the WinForms app we work on. This works great, because it gets all the important stuff into a Presenter that we can easily test. After implementing a couple dozen of these presenters, I've notcied common patterns and dependencies emerge within the presenters. Inspired by Ayende's recent string of posts on limiting abstractions, I decided to revisit what we do in the presenters, what sorts of infrastructure they provide, and how we can make it faster and easier to create new presenters.

The first thing I'm looking at is NHibernate. I've tried numerous means of abstracting it away (PersistenceConversation, Reporsitories, etc.). They all help with testing, but at the expense of application performance. I've finally convinced myself that they're not providing the value they should. So, I've decided to embrace the fact that I'm using NHibernate, and take full advantage of all the facilities it provides. That, of course, means we're coupled to a database. 

## Testing with NHibernate In-Memory

Testing against a SQL Server database is slow. SQLite is fast. SQLite in-memory is really fast. So fast, you can recreate all the tables in your db from your mapping files, ensuring a blank slate for your tests. Unfortunately, SQLite creates a new, blank database everytime you open a new session or stateless session. That is, each session opening seems to open a new IDbConnection, which resulted in a blank db. I could not, for the life of me, figure out how to get connection pooling to happen. Maybe it's not supported with SQLite.
 
This is an issue for two reasons:

1.  I don't want my presenters to depend on a single session. This is just a bad idea. Why? When an exception is thrown in a session, it puts the session into an unknown state, and stuff stops working. You then have to dispose of the session and create a new one. Very tough to do if you don't have access to the SessionFactory...

2.  As I mentioned, SQLite appears to open a new in-memory database every time you grab a new session or stateless session from the session factory. That's utterly useless because any entities you've persisted in the Arrange part of the test are GONE by the time you get to the Act and Assert portions.
 
So, a thin abstraction over the SessionFactory is in order. Fluent NHibernate has one called ISessionSource. It comes in two flavors, one for production, and one aptly called SingleConnectionSessionSourceforInMemorySQLiteInMemoryTesting. This worked great, untill I wanted to use a stateless session in one of my presenters. ISessionSource, as FNH implements it, does not expose a method to get a stateless session. Sure, it exposes the ISessionFactory, but for the reasons above, that doesn't work in my test suite.
 
Fortunately, ISessionSource is a very simple abstraction to implement. It simply wraps an ISessionFactory, and exposes CreateSession and CreateStatelessSession methods. 

<script src="https://gist.github.com/2258967.js?file=ISessionSource.cs">
</script>

The production implementation simply delegates to ISessionFactory.OpenSession() and ISessionFactory.OpenStatelessSession(). 

<script src="https://gist.github.com/2258967.js?file=SessionSource.cs">
</script>

The implementation for tests is a bit more involved. CreateSession is the same as FNH's test session source implementation. CreateStatelessSession internally calls CreateSession, which, since you've used it to create your db schema from your mappings, should already be open. It then grabs the IDbConnection from the session, and uses that to open the stateless session. 

<script src="https://gist.github.com/2258967.js?file=SQLIteInMemorySessionSource.cs">
</script>

This ensures that both will use the same connection, and thus the same database.

The downside to this, of course, is that you're not only still hitting a database, you're also building a SessionFactory (expensive!), as well as a database. The former is done once per test assembly. The latter is done every test. The up side is that all your queries and commands are tested. This probably makes these integration tests as opposed to unit tests. I have some thoughts on how to get them back to fast unit tests, inspired by this Ayende post: [Limit your abstractions: And how do you handle testing?](http://ayende.com/blog/154273/limit-your-abstractions-and-how-do-you-handle-testing).
