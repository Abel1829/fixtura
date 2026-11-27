# Fixtura

Fixtura is a full-stack tournament management system for football, built for a local tournament organizer. Organizers create tournaments in a league or knockout format and generate the match schedule, referees record what happens in each match, team managers look after the players of their team, and spectators follow scores, tables and brackets on a public website without logging in.

I am building Fixtura as two architectures that solve the same problem, so that I can compare them:

1. **A monolith**: one Spring Boot application with one PostgreSQL database and a React website.
2. **A distributed version**: the same system, with the processing that happens after a match is finished moved into a separate worker program. The application and the worker communicate through RabbitMQ and an Apache Camel route.

The reasons for this choice are written down in [decision record 001](docs/decisions/001-monolith-first-then-distributed.md).

## Status

The monolith is under construction. The database and the Spring Boot application start and connect to each other. The features are added one small piece at a time, and the documents in `docs` are updated together with the code.

## Technology

| Part | Technology |
|---|---|
| Server application | Java 21, Spring Boot, Gradle |
| Database | PostgreSQL 16 |
| Website | React (planned) |
| Messaging for the distributed version | RabbitMQ, Apache Camel (planned) |
| Running the system | Docker and Docker Compose |

## Repository layout

```
fixtura/
  backend/           Spring Boot application (the monolith)
  branding/          Logo files and icons
  docs/
    architecture/    Characteristics, C4 diagrams, domain model
    decisions/       Decision records
  docker-compose.yml PostgreSQL database for development
```

More folders (`frontend`, `worker`) are added when those parts are built.

## How to run it

### What you need

- Java 21
- Docker Desktop (it must be running)
- Git

### 1. Get the code

```
git clone https://github.com/Abel1829/fixtura.git
cd fixtura
```

### 2. Start the database

```
docker compose up -d
docker compose ps
```

`fixtura-db` should be running and, after a few seconds, healthy. The database is available on port 5434 of the computer. The development password is `fixtura_dev`, and it is meant for local use only.

### 3. Start the application

```
cd backend
.\gradlew.bat bootRun
```

On macOS or Linux, use `./gradlew bootRun`. The first start downloads the libraries and takes a few minutes. The application listens on port 8081.

### 4. Check that it works

Open this address in a browser:

```
http://localhost:8081/actuator/health
```

The answer contains `"status":"UP"` and a `db` entry that is also `UP`. This means that the application is running and that it can reach the database.

### Run the tests

```
cd backend
.\gradlew.bat test
```

The database must be running, because the tests start the application.

### Stop everything

Press `Ctrl+C` in the terminal where the application runs, then:

```
docker compose stop
```

## Documentation

| Document | Content |
|---|---|
| [Architecture characteristics](docs/architecture/01-architecture-characteristics.md) | The qualities Fixtura must have, ranked, with measurable targets |
| [C4 context](docs/architecture/02-c4-context.md) | Fixtura and its users |
| [C4 containers](docs/architecture/03-c4-containers.md) | The containers of the monolith and of the distributed version |
| [Domain model](docs/architecture/04-domain-model.md) | The entities, their relationships and the rules of the system |
| [Decision record 001](docs/decisions/001-monolith-first-then-distributed.md) | Why there are two architectures |
| [Decision record 002](docs/decisions/002-scope-local-tournament.md) | Why the scope is a local tournament |
