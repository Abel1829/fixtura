# Architecture characteristics of Fixtura

Author: Abel Zewdie Menkir

Last updated: 25 November 2026

## 1. Purpose of this document

Before I write any code, I decide which qualities Fixtura must have. In software architecture these qualities are called architecture characteristics. A characteristic is not a feature. A feature is something the system does, for example "generate a league draw". A characteristic describes how well the system must do it, for example "generating the draw for 64 teams takes less than ten seconds".

I cannot make every quality perfect at the same time, because improving one often hurts another. A very flexible system is usually harder to keep simple, and a very scalable system is usually harder to keep consistent. So for each characteristic I choose a priority (High, Medium or Low), a measurable target and the requirements that motivate it. The priorities are the base for every later architecture decision.

The requirements give no figures, so all numeric targets in this document are my own assumptions. I chose them to be realistic for a tournament system of normal size and to be checkable later.

## 2. The system in short

Fixtura is a tournament management system for football. It has four kinds of users:

- The **organizer** creates tournaments, adds pitches, teams, referees and team managers, and generates the match schedule.
- The **referee** records what happens during a match (goals, cards, substitutions) and submits the final result.
- The **team manager** manages the players of one team and chooses the line-up for each match.
- The **spectator** opens the website without logging in and follows scores, tables and news.

Fixtura is built as two architectures that solve the same problem: a **monolith** (one program with all the rules and one database) and a **distributed version** (several programs that run separately and communicate through messages). Section 7 compares them using the characteristics defined here.

## 3. Requirements

I number the requirements so that each characteristic can point back to the requirement that motivates it.

| ID | Requirement | Description |
|---|---|---|
| R1 | Tournament management | Organizers create and edit tournaments, each with a format (league or knockout). |
| R2 | Pitch management | Organizers create and edit pitches. A pitch is a playing field where a match is scheduled. |
| R3 | Schedule generation | Organizers generate the match schedule. A pitch, a referee or a team cannot be in two matches whose times overlap. |
| R4 | Result registration | Referees record match events. The score is calculated from the goal events. |
| R5 | Teams and line-ups | Team managers manage their players and choose the line-up for each match. |
| R6 | Public access | Spectators see matches, results, standings and news without logging in. |
| R7 | Standings | The league table is calculated from finished matches and never edited by hand. |
| R8 | Roles and ownership | Each user has a role. An organizer sees only their own data. A team manager acts only on their own team. A referee acts only on assigned matches. |
| R9 | Audit | Changes to results, fixtures, match status and users are logged. |
| R10 | Time constraint | The first complete version is due on 30 December 2026. |
| R11 | Initial scope | The first version supports football. |
| R12 | Future sports | The design should allow other sports and rules to be added later. |

## 4. Architecture characteristics

### 4.1 Simplicity: High

Requirements: R10, R11.

The main users are four groups with probably a few hundred people at peak. A system that is hard to understand would cost me some time, and it would be hard to explain and defend.

Target: the first architecture is one deployable program with one database. Someone who follows the documentation can run the whole system locally in less than 30 minutes.

### 4.2 Modularity: High

Requirements: R1 to R9, R12.

Fixtura has several areas with different responsibilities: identity and access, pitches, teams and players, tournaments and draw, matches and results, standings, news, and audit. They interact, but they must not become one tightly connected block. Clear boundaries also make it possible to move one flow into a separate program in the distributed version.

Target: six to eight modules. Each module exposes only the operations the others need. There are no cyclic dependencies between modules, and an automatic test checks this. A change to knockout progression does not require changes to registration or the public pages.

### 4.3 Maintainability: High

Requirements: R3, R4, R7.

The competition rules are the part most likely to change as I understand them better: tie-break order, line-up limits, draw details, eligibility. In this project, maintainability means changing or correcting an existing rule. Adding new sports is a different quality, evolvability, described in 4.6.

Target: a small change to an existing rule can be implemented and verified in one working day. Each rule lives in one identifiable place in the code and is documented there.

### 4.4 Testability: High

Requirements: R3, R4, R7.

A wrong rule is visible to everyone. A mistake in the conflict check double-books a referee, and a mistake in the points calculation changes the table. The cases that deserve most attention are conflict detection, schedule generation, score derivation, standings and knockout progression.

Target: at least 80% automated test coverage of the core business logic. Coverage alone proves little, so each critical rule also has explicit tests, for example an odd number of teams in a round-robin, a tie between two teams in the table, a goal removed from a match, and a winner advancing through every knockout round.

### 4.5 Deployability: Medium

Requirements: R10 (indirectly).

I will change and release the system many times. Slow releases would waste my limited time. I rate it Medium because a single deployable program already keeps deployment simple in the first architecture. In the distributed version it matters more, because several programs and a message broker must start together.

Target: the whole system starts with one command using Docker Compose. A routine release takes about 15 minutes. The previous working version can be restored within 30 minutes if a release fails.

### 4.6 Evolvability: Medium

Requirements: R11, R12.

Fixtura starts with football, but other sports may use different scoring, a different ranking procedure, or no draws. The risk is that football assumptions spread across the whole code. With my time limit I do not implement other sports. I only keep the sport-specific rules in one place, so that a second sport would not require rewriting the rest.

Target: scoring, ranking and eligibility rules are isolated in the rules' module. Adding a new tournament format does not require rewriting registration, result recording or the public pages. Existing tournaments and their results stay valid after such a change.

### 4.7 Responsiveness: Medium

Requirements: R3, R4, R6.

Spectators (Visitors) expect scores and tables to load quickly, and organizers and referees expect schedule generation and result entry to be fast. There are no strict real-time needs.

Target: with up to 100 concurrent users, 95% of ordinary read and update requests complete within two seconds. Generating a round-robin schedule for up to 64 teams (2,016 matches) takes less than ten seconds.

### 4.8 Scalability: Medium

Requirements: R6.

The expected use is moderate, but success could mean several tournaments running in parallel.

Target: the system handles up to 10 active tournaments, about 1,000 registered teams in total and 100 concurrent users without changing the architecture.

### 4.9 Elasticity: Low

Requirements: R6.

The most realistic burst is visitors checking standings during a final. That is real but short, and it does not justify automatic scaling.

Target: no autoscaling. The system absorbs a short peak of about three times the normal load (roughly 300 concurrent visitors) with 95% of requests answered within five seconds. Caching the standings and result pages is enough for this.

### 4.10 Fault tolerance: Medium

Requirements: R3, R4.

If the system fails during a tournament, referees cannot enter results and teams cannot see their schedule. The worst outcome is losing a confirmed result. Continuous operation despite every failure is not required.

Target: a confirmed match result is never lost after it has been successfully saved. The data is stored in a durable database with regular backups. The recovery time objective (RTO), the time to restore service after a serious failure, is at most one hour. The recovery point objective (RPO), the amount of data that may be lost, is zero for confirmed results.

### 4.11 Overall cost

Requirements: R10.

Overall cost here means the effort required to build the system. With one developer and a fixed deadline, any extra architectural complexity must be justified by a concrete need. I avoid infrastructure that the requirements do not demand, and the distributed version moves only one flow out of the monolith.

For partitioning, I chose domain partitioning. The business areas (tournaments, teams, scheduling, results, standings) are distinct, and the possibility of other sports makes domain boundaries more useful than technical layers. Layers can still exist inside each module, so the two approaches complement each other.

## 5. Additional characteristics

These qualities are not part of the main list above, but Fixtura needs them because of its data and its users.

### 5.1 Data consistency: High

Requirements: R3, R4, R7.

The data must never contradict itself. A pitch, a referee or a team cannot be in two overlapping matches. The score is calculated from the goal events, so the two cannot disagree. The league table is calculated from finished matches. A shirt number is unique inside a team. If these numbers cannot be trusted, nothing else in the system has value, so this is the most important of the additional characteristics and one of the three that drive the architecture.

Target: all rules are checked on the server. Changes that belong together are saved in one database transaction, which means that either all of them are saved or none is. The database also protects the most important rules, for example uniqueness, as a second safety net.

### 5.2 Security and access control: High

Requirements: R8.

The system stores personal data about players and gives several people different write rights over the same tournament. A mistake in permissions would let one person change data that belongs to somebody else.

Target: users log in and receive a signed token that is sent with every request. Every operation that changes data checks the role and the ownership on the server. Passwords are stored as hashes, a one-way scrambled form that cannot be turned back into the original password. spectators can only read public data.

### 5.3 Auditability: Medium

Requirements: R9.

Results and fixtures can be disputed, so it must be possible to see who changed what and when.

Target: every change to a result, fixture, match status or user is logged with the user, the time, and the old and new values.

### 5.4 Usability: Medium

Requirements: R6.

Most visitors will open the public pages on a phone.

Target: the website works on phones and computers and has a light and a dark theme.

## 6. Prioritization overview

| Characteristic | Priority | Target | Requirements |
|---|---|---|---|
| Simplicity | High | One deployable program, runs locally in under 30 minutes | R10, R11 |
| Modularity | High | 6 to 8 modules, no cyclic dependencies | R1 to R9, R12 |
| Maintainability | High | Rule change in at most one working day | R3, R4, R7 |
| Testability | High | At least 80% core coverage, explicit rule tests | R3, R4, R7 |
| Deployability | Medium | One-command start, release in 15 minutes, rollback in 30 minutes | R10 |
| Evolvability | Medium | Sport-specific rules isolated in one module | R11, R12 |
| Responsiveness | Medium | 95% of requests under 2 seconds at 100 users | R3, R4, R6 |
| Scalability | Medium | 10 active tournaments, 1,000 teams, 100 concurrent users | R6 |
| Elasticity | Low | No autoscaling, 3 times peak absorbed with caching | R6 |
| Fault tolerance | Medium | RTO of one hour, RPO of zero for confirmed results | R3, R4 |
| Overall cost | Constraint | Complexity only where a requirement demands it | R10 |
| Data consistency | High | Rules enforced on the server and in transactions | R3, R4, R7 |
| Security and access control | High | Signed tokens, role and ownership checks | R8 |
| Auditability | Medium | Every important change is logged | R9 |
| Usability | Medium | Phone and computer layout, light and dark theme | R6 |

The characteristics that drive the architecture, the ones that win when two of them conflict, are **data consistency**, **simplicity** and **modularity**.

## 7. Two architectures judged by the same characteristics

| Characteristic | Monolith | Distributed version |
|---|---|---|
| Simplicity | One program, one database, one place to debug | Several programs and a message broker, more to understand |
| Modularity | Modules inside one program | Separate programs |
| Data consistency | Easy to guarantee with one database and one transaction | Harder, because the programs communicate by messages and some data becomes correct a moment later |
| Testability | Rules tested in one place without a network | Messages and timing also have to be tested |
| Deployability | One program and one database to start | Several programs and a broker to start together |
| Fault tolerance | One failure stops everything | A failed worker can be restarted while the rest keeps running |
| Scalability | The whole program is copied | Only the busy part is copied |

I expect the distributed version to be weaker on simplicity, consistency and deployability, and stronger on separation, fault isolation and scalability. Because data consistency and simplicity are my main characteristics, the monolith is the main architecture of Fixtura. The distributed version moves only one flow out of it, the processing that happens after a match is finished. The comparison between the two columns is the base of my final evaluation of the architecture.

Note: If building the distributed version shows that these priorities should change, I will record the change in a new decision record instead of editing this file silently.

