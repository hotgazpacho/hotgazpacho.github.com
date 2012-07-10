---
layout: post
title: "Using the Reactive Extensions with WinForms"
published: false
---
## Background

The project I'm working right now is a WinForms project. In order to make it more testable and esier to change, we've settled on an MVP pattern, implementd as a variation of [the one Mark Nijhof describes](http://elegantcode.com/2009/12/19/using-conventions-with-passive-view/). We've implemented an [Event Aggregator](http://www.martinfowler.com/eaaDev/EventAggregator.html) on top of the [Reactive Extensions](http://msdn.microsoft.com/en-us/data/gg577609.aspx)(RX), [similar to the one José F. Romaniello describes](http://joseoncode.com/2010/04/29/event-aggregator-with-reactive-extensions/), to enable decoupled, messaging between presenter instances. Presenters are responsible to subscribing to the types of events, optionally meeting some condition, that they are interrested in, and then performing some action when such an event arrives. 

A typical scenario that leverages this functionality is a Master-Detail relationship. One presenter displays a list of something. The user can then bring up a Detail view of that item and make changes to it. When the user saves the detail, the detail presenter publishes a message indicating which item has changed. The master presenter, which was subscribed to receive such events, then issues a query to refresh that single item (rather than the entire list) and update the master view.

## The Problem

Now, originally, when we were subscribing to events in the presenter, the event aggregator required that we pass in an instance of an [IScheduler](http://msdn.microsoft.com/en-us/library/system.reactive.concurrency.ischeduler(v=vs.103).aspx). In our case, since this was a WinForms app, we would create an instance of a [ControlScheduler](http://msdn.microsoft.com/en-us/library/system.reactive.concurrency.controlscheduler(v=vs.103).aspx) in the [Shown event](http://msdn.microsoft.com/en-us/library/system.windows.forms.form.shown), and then raise a standard .NET event with this scheduler instance as an argument. We use the ControlScheduler because it handles the Invoke/BeginInvoke stuff for us when we use it to schedule some work to be done* The presenter, which is wired up to receive this event, would then use this instance to pass to the Event Aggregator to subscribe.

Everything *Worked on My Machine&#8482;*, but when our testers got a hold of it, they always received an `InvalidOperationException: Invoke or BeginInvoke cannot be called on a control until the window handle has been created.` Needless to say, this left me scratching my head. We raise our event long after the [Handle](http://msdn.microsoft.com/en-us/library/system.windows.forms.control.handle) had been created. Why did the ControlScheduler instance think that the Handle had not been created?

Well, as it turns out, the Handle that the ControlScheduler had a reference to *had* been disposed. In fact, something in the way this form interacted with the application shell (probably adding it to a TabPage of a TabControl and hiding the ControlBox) had actually caused the win32 Window to be destroyed and recreated at least once. As [Kevin Dente](https://twitter.com/kevindente) pointed out to me:

<blockquote class="twitter-tweet" data-in-reply-to="222317462530703361"><p>@<a href="https://twitter.com/hotgazpacho">hotgazpacho</a> yup. That’s how winforms let’s you change window properties that in win32 requires destroying/recreating the window</p>&mdash; Kevin Dente (@kevindente) <a href="https://twitter.com/kevindente/status/222332927395115008" data-datetime="2012-07-09T14:14:35+00:00">July 9, 2012</a></blockquote>

OK, so we can't depend on an instance of a ControlScheduler being valid all the time. Which means that we can't rely on it in a call to [ObserveOn](http://msdn.microsoft.com/en-us/library/hh211920(v=vs.103).aspx), which our implementation of the Event Aggregator was doing when we called Subscribe on it. In fact, RX guru [Paul Betts](https://twitter.com/xpaulbettsx) hinted as much when I was groping about in the dark on twitter:

<blockquote class="twitter-tweet" data-in-reply-to="221666903297507328"><p>@<a href="https://twitter.com/hotgazpacho">hotgazpacho</a> @<a href="https://twitter.com/mattpodwysocki">mattpodwysocki</a> If you are doing asynchronous stuff, at some point you’ll want to. Call it as little as possible</p>&mdash; Paul Betts (@xpaulbettsx) <a href="https://twitter.com/xpaulbettsx/status/221667340587249665" data-datetime="2012-07-07T18:09:47+00:00">July 7, 2012</a></blockquote>

So, time to refactor!

## The Solution

Once I understood the problem, the solution became evident: any time the underlying Handle on the form changed, we'd need to notify the presenter. That way, when we'd actually need to schedule work to be done, we'd have a valid instance. This also meant that we'd need to forego having the EventAggregator call ObserveOn when we subscribed to events. Finally, when we wanted to interact with the View, we should use [IScheduler.Schedule](http://msdn.microsoft.com/en-us/library/hh229734(v=vs.103).aspx) for complex updates, or ensure that the View performed the necessary switch for BeginInvoke for simple assignments.

### Step 1 - Get a valid IScheduler to the presenter

Fortunately, the Form class has an event to let us know that a Handle has been created, *or recreated*, for the form: [HandleCreated](http://msdn.microsoft.com/en-us/library/system.windows.forms.control.handlecreated.aspx). The documentation is a little misleading, as HandleCreated is raised not only when the form is first Shown, but anytime that the Handle is created. Instead of creating a one-off ControlScheduler in the form's Shown event, we'll add an IScheduler property to the form:

    public IScheduler Scheduler { get; private set; }
    
Then, we hook into the form's HandleCreated event so we can create a new, valid ControlScheduler instance when we need to:

    HandleCreated += (s,e) {
        Scheduler = new ControlScheduler(this);
        OnReady.Raise(Scheduler);
    };

### Step 2 - Stop Subscribing with ObserveOn

Pretty self-explanatory. Remove the call to ObserveOn in the Event Aggregator's Subscribe method:

    public IDisposable Subscribe<TEvent>(Action<TEvent> onEvent) {
        return GetEvent<TEvent>().Subscribe(onEvent);
    }

### Step 3 - Use !Scheduler.Schedule for Complex UI updates

Typically, you can use an extension method, [like the one Derick Bailey recommended to me](http://lostechies.com/derickbailey/2011/01/24/asynchronous-control-updates-in-c-net-winforms/) to ensure that you can safely update UI controls from the non-UI thread. In some cases, though, this is simply not enough. The specific case that caused me trouble was [BindingList<T>](http://msdn.microsoft.com/en-us/library/ms132679.aspx), specifically when it raised [ListChangedEvent](http://msdn.microsoft.com/en-us/library/ms132742.aspx). This could potentially happen on a background thread, which would result in cross-thread exceptions. I solved this by setting [RaiseListChangedEvents](http://msdn.microsoft.com/en-us/library/ms132728.aspx) to false for my binding list, updating the list, then using the IScheduler instance to raise ListChangedEvents on the UI thread:

    _scheduler.Schedule(() => _model.SomeBindingList.ResetBindings());
    
And there you have it! I hope someone else finds this useful, since most people using RX seem to be doing it with WPF and not WinForms.

<script src="//platform.twitter.com/widgets.js" charset="utf-8"></script>