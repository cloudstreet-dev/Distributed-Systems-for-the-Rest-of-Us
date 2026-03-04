# Distributed Systems for the Rest of Us

*A CloudStreet Book*

---

This book is for developers who have hit a race condition in production and lost.

Not "lost" as in "my program crashed and I debugged it." Lost as in: "The data was wrong, nobody knows how it got wrong, two customers called, and we spent a Thursday morning reading logs from three different services trying to reconstruct what happened. We found the bug. We fixed it. We added a comment that says `// THIS ORDER MATTERS`. We moved on."

That experience — that specific Thursday — is what distributed systems is actually about.

The academic literature calls it "the problem of coordinating state across multiple nodes in the presence of partial failures." You called it "the worst day this quarter." Both descriptions are accurate.

---

## What This Book Is

This is a practical book about distributed systems concepts: the ideas you need to understand *why* things go wrong and *how* to design systems that fail less badly.

It is not a comprehensive academic treatment. It will not teach you to implement Raft from scratch (though you'll understand how Raft works). It will not make you ready to design Google Spanner (though you'll understand what Spanner is solving). What it will do is make you a better architect, a sharper debugger, and someone who can look at a system design and say "that's going to have consistency problems under partition" before it's 3am and the on-call rotation has your name on it.

The concepts here scale from "I have two microservices and a database" all the way up. The math works at any scale. The trade-offs just get more expensive as you grow.

---

## What This Book Assumes

You understand databases and networking at a working level. You know what a transaction is. You've dealt with HTTP. You've probably seen a queue. You don't need reminding what a foreign key is.

What this book does *not* assume is that you've thought deeply about what happens when your database has two replicas and the replication lag is 300ms and a request hits the replica and not the primary. Or what "eventually consistent" actually means in terms of what your users see and when.

That's the gap. This book lives there.

---

## How to Read This Book

Sequentially, ideally. The chapters build on each other. Fallacies before CAP, CAP before consistency models, consistency before replication. If you skip around, the forward references will be annoying.

That said: if you're here because you're debugging something *right now*, jump to the chapter that matches your problem. The chapter on network partitions doesn't require you to have read the chapter on clocks. Use what you need.

---

*Let's start with the uncomfortable truth: you're already doing this.*
