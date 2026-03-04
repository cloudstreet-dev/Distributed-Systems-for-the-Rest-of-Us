# Distributed Systems for the Rest of Us

*A CloudStreet Book*

A practical guide to distributed systems for developers who aren't at Google scale — but still need to think like they are.

This book is for working developers who have hit a race condition in production and lost. Not lost as in "my program crashed." Lost as in: two customers called, nobody knows how the data got wrong, and you spent a Thursday morning reading logs from three different services.

**[Read online →](https://cloudstreet-dev.github.io/Distributed-Systems-for-the-Rest-of-Us/)**

---

## Table of Contents

- [Preface](src/preface.md)
- [Chapter 1: You're Already Doing Distributed Systems](src/introduction.md)
- [Chapter 2: The Eight Fallacies of Distributed Computing (They Were Right)](src/fallacies.md)
- [Chapter 3: CAP Theorem: Consistency, Availability, and the Lie You Have to Pick](src/cap.md)
- [Chapter 4: Consistency Models: From Strong to Eventual and Everything Uncomfortable In Between](src/consistency.md)
- [Chapter 5: Replication Strategies: Leaders, Followers, and Who Gets to Be Right](src/replication.md)
- [Chapter 6: Consensus: How Nodes Agree on Anything at All](src/consensus.md)
- [Chapter 7: Time Is a Flat Circle: Clocks, Ordering, and Logical Time](src/clocks.md)
- [Chapter 8: Fault Tolerance: Designing for the Failures You Know and the Ones You Don't](src/fault.md)
- [Chapter 9: Network Partitions: When the Network Lies to You](src/partitions.md)
- [Chapter 10: Patterns That Work: Sagas, Outbox, Circuit Breakers, and Friends](src/patterns.md)
- [Chapter 11: Making Real Decisions: How to Think About Your Actual System](src/tradeoffs.md)
- [Chapter 12: Where to Go From Here](src/conclusion.md)

---

## Building Locally

### Prerequisites

Install [mdBook](https://rust-lang.github.io/mdBook/guide/installation.html):

```bash
cargo install mdbook
```

Or via package manager:

```bash
# macOS
brew install mdbook

# Via cargo (any platform)
cargo install mdbook
```

### Build and serve

```bash
# Clone the repo
git clone https://github.com/cloudstreet-dev/Distributed-Systems-for-the-Rest-of-Us.git
cd Distributed-Systems-for-the-Rest-of-Us

# Serve locally with live reload
mdbook serve --open

# Build static output only
mdbook build
```

The built book will be in `./book/`. The local dev server runs at `http://localhost:3000`.

---

## About

Published by [CloudStreet](https://cloudstreet.dev) under Creative Commons CC0.

Technical, accurate, and genuinely amusing. Think dry wit, not dad jokes.

The reader is a competent developer who isn't working at Google scale and probably never will be — but they still need to think like someone who is.
