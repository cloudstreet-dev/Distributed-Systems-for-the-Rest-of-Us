# You're Already Doing Distributed Systems

Here is a thing that happens to developers:

You're building a feature. You need to send a welcome email when a user signs up. Simple enough. You save the user to the database, then call your email service. It works fine in development. It works fine in staging. You ship it.

Three weeks later, you notice some users never got the welcome email. You dig into the logs. The database write succeeded. The email call... timed out. Or the email service was down for 90 seconds at 2am. Or the server process was killed halfway through. The user exists. The email never sent.

Congratulations. You have just experienced your first distributed systems problem in production.

The system was distributed the moment you had two things — a database and an email service — that needed to agree on what happened. They didn't agree. Reality was inconsistent. Someone noticed.

---

## What "Distributed" Actually Means

People hear "distributed systems" and picture Google's infrastructure: hundreds of datacenters, thousands of nodes, petabytes of data, entire teams whose job is keeping the thing running. That's distributed systems at scale. But the *problems* start the moment you have more than one process that needs to coordinate.

Two microservices talking over HTTP: distributed system.
An application server plus a database: distributed system.
A job queue with workers: distributed system.
An application that calls a third-party API: distributed system.

The defining characteristic isn't scale — it's that you have **multiple processes that don't share memory**, connected by a network, that need to do work together.

This matters because:

- **Processes can fail independently.** Your app server can die while the database stays up, or vice versa.
- **The network between them can fail.** Or go slow. Or deliver messages out of order.
- **There's no shared clock.** Two processes saying "this happened at 14:32:07" may be referring to slightly different moments, and may not agree on which happened first.
- **There's no shared state.** Each process sees its own view of the world, and those views can diverge.

If that sounds scary, it should, a little. But these problems are understood. They have names. They have known solutions and known trade-offs. That's what this book is about.

---

## The Problems Have Been Around Forever

The canonical paper on distributed systems fallacies was written in the early 1990s. The CAP theorem was formalized in 2000. Lamport clocks date to 1978. The Byzantine Generals Problem is from 1982.

These aren't new problems. They're not unsolved problems. They're just problems that become *your* problems the moment you wire two processes together.

The frustrating part isn't that the problems exist. It's that they're often invisible until something goes wrong. Single-process applications fail loudly: an exception propagates up, you get a stack trace, you find the line. Distributed systems fail quietly: a write succeeded on one side and silently didn't propagate to the other. A message was delivered twice. A timeout happened and the caller doesn't know whether the operation succeeded.

---

## A Taxonomy of What Can Go Wrong

Before we get into solutions, let's be precise about the problem space. Distributed systems fail in a few distinct ways:

### Crash failures

A process dies. It stops responding. It doesn't send any more messages. This is actually the *best kind of failure* — it's detectable and it's clean. The dead process doesn't lie to you.

### Omission failures

A process is running but stops responding to some (or all) messages. This is harder — from the outside, it looks like a crash, but it isn't. Network partitions produce this kind of behavior.

### Timing failures

A process responds, but too slowly. Your distributed system makes assumptions about timing (implicit or explicit), and a slow response violates them. This is why timeouts are hard: too short and you get false failures; too long and you're just sitting there waiting while your users notice.

### Byzantine failures

A process responds with incorrect or malicious information. It actively lies. This is the hardest category to handle and also the rarest in systems you control. We'll touch on it when we discuss consensus, but for most of this book, we assume nodes fail by crashing or going silent, not by fabricating responses.

Most production incidents involve omission failures and timing failures, not Byzantine ones. Your database isn't lying to you. It's just slow, or unreachable, or in the middle of a failover.

---

## The Fundamental Tension

Every distributed systems decision comes down to a tension between two things:

1. **Correctness**: making sure your system's data and behavior are accurate and consistent
2. **Availability**: making sure your system keeps working even when parts of it fail

These goals pull in opposite directions. To be maximally correct, you sometimes need to stop and refuse to answer (because you're not sure you have the right answer). To be maximally available, you sometimes need to answer even though you might be wrong.

Most systems don't live at either extreme. They make a choice somewhere in the middle, often without realizing they've made a choice at all. One of the goals of this book is to help you make that choice deliberately.

---

## What "Thinking Like a Distributed Systems Engineer" Means

It mostly means getting comfortable with a specific mental shift: **instead of thinking about what your code does, think about what can happen between the steps.**

Single-process thinking: "I write to the database, then I send the email."

Distributed-systems thinking: "Between writing to the database and sending the email, what can fail? What does the system look like if it fails? Is that state valid? Can we recover from it?"

This isn't pessimism. It's precision. The failures will happen. The question is whether you designed for them or not.

The email example at the top of this chapter has several correct solutions. You could use a transactional outbox pattern (write the email intent to the database in the same transaction as the user creation, then have a separate process pick it up and send it). You could use an event-driven architecture where "user created" is an event that the email service consumes reliably. You could accept the rare failure and add a retry mechanism.

All of these are reasonable. None of them are "just call the email API in the handler." Because that's not the shape of the problem.

---

## Let's Map the Terrain

Here's what the rest of this book covers, and why it's in this order:

**Chapter 2 — The Eight Fallacies** lays out the wrong assumptions that every developer makes about distributed systems. Read them once, internalize them, and a lot of later bugs will become obvious.

**Chapter 3 — CAP Theorem** introduces the central trade-off of distributed systems: you can't have consistency and availability both, all the time. This gives us a framework for making decisions.

**Chapter 4 — Consistency Models** goes deeper on what "consistent" even means. Turns out it's a spectrum, and where you sit on that spectrum has concrete effects on what your users see.

**Chapter 5 — Replication** covers what happens when you have multiple copies of data and need them to stay in sync. Spoiler: they won't, perfectly, and that's okay if you understand the rules.

**Chapter 6 — Consensus** explains how distributed systems make collective decisions — electing leaders, agreeing on a log of operations. This is where Raft and Paxos live.

**Chapter 7 — Clocks and Time** tackles the uncomfortable truth that time doesn't work the way you think it does across a network. Lamport clocks. Vector clocks. Why "last write wins" is more complicated than it sounds.

**Chapter 8 — Fault Tolerance** covers the patterns for building systems that degrade gracefully: retries, idempotency, bulkheads, graceful degradation.

**Chapter 9 — Network Partitions** goes deep on the specific failure mode that causes the most trouble: the network split where both sides are running and neither can tell the other is still there.

**Chapter 10 — Patterns** is the practical chapter: sagas, the outbox pattern, circuit breakers, and other concrete solutions to recurring problems.

**Chapter 11 — Making Real Decisions** brings it together. Given your actual system, your actual scale, and your actual constraints, how do you pick the right tools and make the right trade-offs?

**Chapter 12 — Where to Go From Here** is resources, further reading, and the honest acknowledgment that this book is a starting point, not an ending one.

---

Let's get started with the assumptions that will kill you if you don't unlearn them.
