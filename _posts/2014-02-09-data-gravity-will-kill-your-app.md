---
layout: post
title: "Data Gravity Will Murder Your App"
category: articles
image:
  feature: IMG_20130209_153507.jpg
---

I recently stumbled across [Dave McCrory's blog post in which he coined the term "data gravity."](http://blog.mccrory.me/2010/12/07/data-gravity-in-the-clouds/) Here's the crux of it:

> Consider Data as if it were a Planet or other object with sufficient mass.  As Data accumulates (builds mass) there is a greater likelihood that additional Services and Applications will be attracted to this data. This is the same effect Gravity has on objects around a planet.  As the mass or density increases, so does the strength of gravitational pull.  As things get closer to the mass, they accelerate toward the mass at an increasingly faster velocity.

Mr. McCrory is talking about capital-D Data writ large on the macro scale, but I've found that thinking about data as having gravity within a single application can be a useful metaphor for understanding and avoiding some growing pains that every application faces in its time. As Mr. McCrory says, data has a tendency to attract other data, services, and applications. Even at the application level, data has a certain force that draws in more and more code as it grows. If you don't keep any eye on it, it can trip you up in two very concrete but different ways.

## TL;DR

The first way data gravity will fuck your shit up is in the "classic" start-up good-problem-to-have the-database-is-falling-over sort of way. Simply put, if you have a particular table that grows and becomes all important in your app, soon more and more operations in your app are going to either rely on reading from it it or writing to it. If there's some table that's involved in literally every request to your web app, that's the table where you're going to hit scaling problems of this variety.

The second way data gravity will kill your app has nothing to do with the size of your database or the number of EC2 instances you have spun up behind your load balancer but it's actually far more pernicious. As a particular table grows, as more and more code is coupled to that table, changes in the way that data is used become more and more difficult to reason about.

Once you've spotted data gravity in your app, you have undoubtedly spotted some tightly coupled components in the code as well. This is a drag on your scaling, to be sure, but not in the fun good-problem-to-have ways that make for alluring war stories you can tell investors. Scaling up the team, improving existing behavior, adding features -- data gravity makes all of these tasks more difficult.

These two problems are separate concerns but they're tangled up together. For example, the harder it is to reason about your app because of how tightly wound around this particular table all your logic is, the harder it will be to cache things reliably.

That's the TL;DR of how data gravity will fuck your shit up but so far we've only spoken in very general statements. Let's look at an (admittedly contrived) example and see if we can start thinking of data gravity more concretely.

## What does data gravity look like in the wild?

Let's say you're creating a shopping cart for a site where inventory is likely to sell out, e.g. a ticket sales site where users need to "lock in" their ticket while filling out the rest of their order information.

You start out innocently enough with a `carts` table and a `tickets` table. When a customer adds a ticket to their cart, you create a record in the join table, which we might call `ticket_selections` for now. When it's time to check if a ticket is sold out, you simply tally up all the `ticket_selections` entries and compare the count with the number of allocated stock.

Already the `ticket_selections` table is doing double duty: it's responsible for tracking a particular user's shopping cart as well as the ticket inventory for a particular event. This becomes a problem when tickets for Stevie Nicks' latest tour go on sale and rabid Fleetwood Mac fans start slamming your ticket sales system trying to secure their seats.

Every time a user adds a ticket to their shopping cart we write a new row to the `ticket_selections` table and every time a user checks the availability of an event there's a read across the very same table. All those writes slow down the indexing of the table, making the reads sluggish. Each subsequent event has an amplifying effect on your bottleneck.

Okay, so this is not ideal but you've probably seen worse. However if you let this kind of data gravity go unchecked things can get hairy very quickly. What happens when the team decides to just flag `ticket_selections` as "ordered" so as to keep inventory and purchases in sync. Now the table is responsible for even more business logic, more code touches it, it attracts more data.

What happens when some intrepid junior dev is tasked with adding some feedback mechanism. Customers need to be able to submit feedback about their ticket-purchasing experience and maybe they're prompted to fill out a survey at a particular interval of time after their purchase.

Our young dev might take a look at the tables in our database and think, "Ah ha! There's already a table that links a customer to a ticket they purchased, I'll just tack a feedback table on here and be done!" The dev writes some code that queries the `tickets_selections` table, filters them by date, and then displays a notification to a returning customer to fill out some feedback.

Now our `ticket_selections` table is THE canonical reference for 1) a customer's shopping cart selections, 2) the remaining inventory for a particular event, 3) a customer's orders 4) figuring out whether or not a customer has any feedback to enter, 5) the link between a piece of feedback and the ticket/event/customer it was connected with.

This is what we mean when we talk about "data gravity." The more things that rely on a particular set of data, the more it draws in more data and even more things start to rely on it.

Performance issues aren't the worst part about data gravity though; they're just the most concrete. In our contrived example, maybe caching could mitigate a lot of the headaches our junior dev has introduced. The most pernicious thing about data gravity is that it makes it conceptually much more difficult to reason about changes in the codebase. It means that new requirements around inventory handling have to account for how feedback works or how the past orders are tracked. Suddenly your system is very brittle and some very expensive consultants are recommending you re-write the whole thing in Scala.

This is the real danger of data gravity -- or conversely, this is where the real value in thinking of data as having gravity comes into play. When you encounter a mass of data pulling in all kinds of other data you have surely encountered some flawed design choices. Data gravity problems are symptoms of objects with too many responsibilities.

The lesson is this: database tables are cheap. Giant tables are expensive. Keep your data small and atomic.

## The takeaway
 - Data gravity applies even at the table level
 - Data attracts data, services, and applications but also code and application logic
 - Data gravity is a useful way of thinking about modeling/design
 - Data gravity can be dangerous because it causes database bottlenecks/load issues
 - Data gravity makes reasoning about change difficult

