---
layout: post
title: How do we make complex software less costly?
---

Recently, we have been looking for a new backend developer to join Evojam. I take an active part in the recruitment process, especially in job interviews. What I like best about the interviews is the part where candidates ask their questions.

One of them got me to write this post — “What is your most complex project right now?”

“Well, it depends on what you mean by complex,” we said at first. Then, we did our best to give a meaningful answer, which was rather based on project scale.

But what is software complexity?

## Complexity patterns

So we will try two different angles. First, let’s separate essential and accidental complexity. Every project should be about a particular business problem. A correct solution solves one without creating more issues for the stakeholders.

After that, we will see how coupling makes software difficult to reason about. Coming up with user passwords that fulfill the safety requirements can be challenging. Now imagine also having to confirm with friends and family that they have never used the same phrase.

## Impact of quality attributes on cost

What are the implications of complexity? It costs time and money. Understanding any business problem is a joint effort of developers and stakeholders. If they jump to suggesting solutions too fast, it is easy to confuse the question with one of the possible answers.

Most developers prefer fighting self-imposed technical issues to learning expert domain knowledge. They feel more comfortable dealing with accidental complexity than with what is essential.

As we will see, the essential/accidental and module coupling angles overlap. High accidental complexity stems from a poor understanding of the domain. Tight coupling produces more bugs and slows down the only certain thing in software — change.

## Timing is everything

Before any code gets written, we are looking at essential complexity. This is what the stakeholders expect the system to do. We all meet to write down the requirements and get into more detailed scenarios. As time goes on, things become hard to explain in plain text, so we start drawing diagrams. At this stage, we need to understand why things should work in a certain way.

Once we start implementing, there is no way not to introduce accidental complexity. In Java, we have to handle null values. In a relational database, we can only go so far without schema migration scripts. In any distributed architecture, remote call failures are sure to happen.

Modern technologies help us deliver working software faster. Implementing most systems in low-level languages would not do any good. Still, all technologies and patterns suit certain essential complexities but come at a price. We need to stay aware of the complexities that we introduce ourselves and the cost thereof.

## Interdependence in software architecture

I recently came across the following definition. It was in the context of financial systems. Still, it bears a striking resemblance to challenges of software design:

*'A complex domain is characterized by the following: there is a great degree of interdependence between its elements, both temporal (a variable depends on its past changes), horizontal (variables depend on one another), and diagonal (variable A depends on the past history of variable B).'* - Nassim Taleb

Let me try to illustrate this with an example. Take the Comedy Agency used in my post on temporal object patterns using JPA. The stakeholders want comedians to refine their jokes and store joke versions data. In its simplest form the application accepts any new joke version:

![image](https://user-images.githubusercontent.com/49154776/121803802-7a337580-cc43-11eb-83e2-7c6e4104a459.png)

For this and the following diagrams, I used free online software — SequenceDiagram. It translates what you draw into plain text, letting you share it with the rest of the team, as well as put in version control:

```
title Low complexity

participant Comedian
participant Application

Comedian->Application:Update joke #9
Comedian<--Application:Joke updated
```

Coming back to the earlier definition, when does this example become more complex? For example, if the expectation is for joke versions not to repeat. The application would then need to store all past versions and read them before any write. This leads to more code as well as a need to test both possible scenarios:

![image](https://user-images.githubusercontent.com/49154776/121803819-90d9cc80-cc43-11eb-8f93-82d42f0d2b7d.png)

![image](https://user-images.githubusercontent.com/49154776/121803832-98997100-cc43-11eb-8a1b-dc2fffe252a5.png)

This covers temporal interdependence, and I wish we could stop there. Effective module boundaries help reduce horizontal and diagonal relations. Unfortunately, it is not very common for domain elements not to interact at all. Teams within organizations exchange information. According to Conway's Law, the systems we design tend to resemble such organizational structures.

In the Comedy Agency example, the competitive advantage for the stakeholders lies in data analysis. We gather audience reactions to jokes to see which comedian told the joke best. We also want to see which joke versions clicked.

In that case, comedians may need to reverse changes they made to jokes. Only successful joke versions are available for restore. What if joke contents and reactions are the responsibilities of two different server-side modules? Now, these modules of the application need to exchange historical data:

![image](https://user-images.githubusercontent.com/49154776/121803842-a949e700-cc43-11eb-93b8-7abfb48f3435.png)

And this is just one of the possible scenarios! You can almost smell all the bugs, they are right around the corner. Of course, database access can also fail. The call from Jokes to Reactions can time out. And the logic to determine if reactions were good — there is no way the initial version is final.

## Service-oriented business analysis

Now, how can we use this to our advantage? In any project, a clear idea of essential complexity is invaluable. Without it, we forget about the core problem that someone is paying us to solve. We focus on secondary technical aspects. Take your business problem and the solution. Weigh them against each other. If it is the latter that seems more complicated, I bet you can do a better job.

A solid understanding of your business domain is also a plus if you seek loose coupling. How else do you divide and conquer? The more naive your element boundaries are, the more horizontal dependencies they will produce. These issues will cost you less if you find out earlier. Changes in implementation take more time as your system grows.

## No code software design

We have established that figuring out essential complexity is key. What do we get there?

One technique we like in Evojam is Event Storming. We recently did a live session based on one of Neal Ford's Architectural Katas. This is how Event Storming helped us visualise a short project description:

![image](https://user-images.githubusercontent.com/49154776/121803852-b1098b80-cc43-11eb-846a-1d89f489a27d.png)

Orange notes represent events relevant to the system, like completing a payment. All pending questions are in red. The ordering of notes is relevant because it shows the passage of time in the system. Some things can only happen after something else happens.

Our session took around ninety minutes. We managed to lay out our initial idea of the requirements. If you look at the number of red notes, we still had plenty of questions for the stakeholders.

This is great because we need to track down all unknowns early on. This way we will take more things into account in our design. And yes, you guessed it — cut down accidental complexity.

## Project quality attributes

“What is your most complex project right now?”

This was the original question from one of the candidates. Let me have another go at answering it:

“So I would say all projects are complex. It is crucial that we never stop studying the domain. If we focus on that and keep reassessing our solutions, the chances are we'll stop projects from decay.”

Or another way to put it — we don’t do easy projects, but we sure make them seem so.
