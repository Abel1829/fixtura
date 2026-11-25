# Decision record 001: build a monolith first, then a distributed version

Status: accepted
Date: 25 November 2026

## 1. Context

Fixtura is a tournament management system for football. In `docs/architecture/01-architecture-characteristics.md` I ranked its qualities. The three that drive the architecture are data consistency, simplicity and modularity. The most important rules of the system are about consistency: a pitch, a referee or a team can never be in two overlapping matches, the score is calculated from the goal events, and the league table is calculated from finished matches.

I want to build Fixtura as two architectures that solve the same problem, so that I can compare them honestly:

- a **monolith**, which is one program that contains all the rules and uses one database;
- a **distributed version**, which is several programs that run separately and communicate through messages.

My constraints are fixed. I work alone, about two hours a day, and the first complete version is due on 30 December 2026. That is roughly 70 hours in total, and the frontend (React) is new to me. Any decision has to fit inside this budget.

## 2. Decision

I build Fixtura in two stages.

**Stage 1: a modular monolith.** One Spring Boot application written in Java 21, with one PostgreSQL database and a React website. The code is divided into modules (identity and access, pitches, teams and players, tournaments and draw, matches and results, standings, news, audit). Modules talk to each other only through small public interfaces. The whole system runs locally with Docker Compose. Almost all of the work and all of the business rules live here.

**Stage 2: a small distributed version.** I do not split the whole system. I move only one flow out of the monolith: what happens after a match is finished.

1. When a referee confirms a result, the main application saves it and publishes a "match finished" message to RabbitMQ. RabbitMQ is a message broker. It works like a post office between programs: one program leaves a message and another picks it up later.
2. A separate worker program receives the message through a route defined with Apache Camel, which is a library for describing how messages move between systems.
3. The worker recalculates the standings and writes the audit record.

The distributed version also runs with Docker Compose, and the two versions share the same domain rules.

At the end I compare the two versions using the characteristics from the characteristics document: simplicity, consistency, deployability, fault tolerance, scalability and so on.

## 3. Alternatives I considered

**A. Monolith only.** It is the cheapest and the safest option. I rejected it because it cannot show how the same problem behaves in a distributed system, and the comparison between two architectures is the main thing I want to learn and show.

**B. A monolith with a message queue inside the same program.** It would use messaging, but everything would still run as one program. I rejected it because it is still a monolith with extra parts, so it would not give me a real distributed architecture to compare.

**C. Full microservices.** Separate services for identity, tournaments, matches and the public pages, each with its own database. It would be the most complete distributed design. I rejected it for three reasons:
- Each service needs its own database, its own security handling and its own deployment. For one developer in 70 hours, the frontend or the tests would be left unfinished.
- The consistency rules (conflicts across pitches, referees and teams) would become problems that span several databases. That works against my first characteristic.
- The result would be much harder to explain and defend than a smaller design that I fully understand.

## 4. Consequences

**Good:**
- The monolith protects data consistency in the simplest possible way. A rule check and the change it allows happen in one database transaction, so they either both succeed or both fail.
- The system stays small enough for me to understand every part and to explain it.
- Because the monolith is modular, moving one flow into a worker is a limited change and not a rewrite.
- The comparison between the monolith and the distributed version is concrete, because both solve the same problem with the same rules.

**Costs and risks:**
- In the distributed version, the standings are no longer updated in the same transaction as the result. The table becomes correct a moment after the result is saved. I accept this delay for standings, and I never apply it to the result itself, which is saved durably before the message is published.
- The distributed version needs more to start and to monitor: a broker and a worker besides the main application and the database. Deployability and simplicity are weaker there.
- A message can be delivered twice or arrive late. The worker must therefore handle the same message more than once without producing a wrong table. I will make the standings calculation repeatable, so that running it twice gives the same result.
- Stage 2 starts on 23 December 2026. If stage 1 is not perfect by then, I still start stage 2 on that date and finish stage 1 inside the remaining time. Both architectures are needed for the comparison.

## 5. Related documents

- `docs/architecture/01-architecture-characteristics.md`: the ranked characteristics that this decision is based on.
