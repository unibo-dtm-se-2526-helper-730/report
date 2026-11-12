---
title: Development
has_children: false
nav_order: 5
---

# Development

## DVCS

The project is versioned with **Git** and hosted on **GitHub**, in a single repository
(monorepo) that contains the backend, the frontend, the release tooling and the CI
workflows.

### Branches

- `main` holds the released code: every merge into it may produce a new release (see
  [CI/CD](../08-cicd)). It is never changed directly: every change arrives through a pull
  request.
- Work happens on short-lived branches created from `main`, one per change, named after
  the kind of change, with the same prefixes as the commit types: `feat/…`, `fix/…`,
  `test/…`, `docs/…`, `build/…`, `ci/…` (for example `feat/trade-republic`,
  `fix/t102-losses-to-carry`).
- A branch is deleted once its pull request is merged.

### Commit messages

Commits follow **[Conventional Commits](https://www.conventionalcommits.org/)**:

`<type>(<scope>): <description>`


- **type**: `feat` (new feature), `fix` (bug fix), `test`, `docs`, `style` (formatting
  only), `refactor`, `build` (build system, dependencies), `ci`, `chore`;
- **scope**: the part of the system involved, usually a backend package or the frontend:
  `ingestion`, `engine`, `reporting`, `api`, `persistence`, `i18n`, `frontend`,
  `backend`, `docker`, `release`;
- **description**: imperative mood, lower case, no final period, saying *what* changes
  (e.g. `feat(ingestion): import Trade Republic CSV exports`).

The convention is not only a matter of style: the release tool reads the types to choose
the next version number (`fix` → patch, `feat` → minor, `!` or `BREAKING CHANGE` → major)
and to write the release notes.

### Pull requests

- Every change reaches `main` through a pull request. Its title is itself a Conventional
  Commit, because the pull request is **squash-merged**: its commits become a single
  commit on `main`, titled like the pull request with the pull request number (e.g.
  `fix(reporting): report the losses to carry forward in T102 (#23)`), and with the list
  of the original commits in the body.
- A pull request groups the commits of one change, split by scope when useful: for
  example the fix in one commit and its tests in another.
- The CI pipeline runs on every pull request (compilation, formatting, static analysis,
  tests on three operating systems): a pull request is merged only when it is green.
- Being an individual project, there is no peer review: each pull request is reviewed by
  the author before merging, with the CI results as the main check.

## Implementation details

### Network protocols

The components communicate over **HTTP**:

- between the browser and the reverse proxy, and between the proxy and the web service;
- the web service talks to the database with the **PostgreSQL wire protocol**, through
  JDBC.

HTTP fits because every interaction is a request from the user that waits for an answer:
there are no updates pushed by the server and no real-time streams, so WebSockets, MQTT or
message queues would bring complexity with no benefit. HTTP is also what browsers speak
natively, and it allows a self-describing API (REST, documented with OpenAPI).

The API is versioned in the path (`/api/v1/...`), so that a future incompatible version
can live next to the current one. Everything runs on the user's machine, so traffic does
not leave it and is not encrypted (plain HTTP on `localhost`).

### Representation of data in transit

- **JSON** for requests and responses: readable, native in the browser and in Spring.
  Field names are in `snake_case`; amounts and quantities travel as **strings**
  (`"390.1"`), not as JSON numbers, so that no decimal digit is lost by floating-point
  conversion in JavaScript.
- **multipart/form-data** for uploading the broker CSV.
- **Problem details** (`application/problem+json`, RFC 9457) for errors: a standard
  format with a status, an error code the client can react to, a message already
  translated, and extra fields when needed (e.g. the list of unclassified instruments).
- The API is described by an **OpenAPI** document, committed in the repository and
  checked by a test against the running application, so that it cannot drift from the
  code.

### Database queries

The database is queried in **SQL**, through **JPA** (Hibernate) and **Spring Data**
repositories:

- simple queries are derived from the method names (e.g.
  `findAllByOrderByExecutedAtAscIdAsc`);
- the few specific ones are written as JPQL queries (e.g. deleting the instruments no
  longer used by any transaction or price);
- the schema is created and changed only by **Flyway** migrations, in plain SQL; Hibernate
  only *validates* that the entities match it at startup, and never changes it.

A relational database and SQL fit because the data is structured and related, the
constraints (foreign keys, check constraints) protect it, and exact decimal types avoid
rounding errors on amounts (see [Design](../03-design)). A NoSQL store would give up
these guarantees without any need for its flexibility.

### Authentication and authorization

There is **no authentication and no authorization**. This is a deliberate choice:

- the application is installed and run by its only user on their own computer, and is
  not exposed on the internet: whoever can reach it is already the owner of the machine;
- there is a single role (the taxpayer), so there is nothing to authorize;
- user accounts, passwords or tokens (OAuth, JWT) would add complexity and new risks,
  such as storing credentials, without protecting anything more.

The limit of this choice is that the application must not be exposed on a network. If it
were offered as a shared online service, it would need user accounts (for example OAuth
with an external identity provider), data separated per user, and HTTPS.

## Technological details

### Languages, frameworks and tools

| Area | Technology | Why |
|---|---|---|
| Backend language | **Java 25** | Strong typing, `BigDecimal` for exact money arithmetic, records and sealed switches for a compact domain model. |
| Backend framework | **Spring Boot 4.1** (Web MVC, Data JPA) | Mature standard for REST services in Java: HTTP layer, dependency injection, transactions and database access with little code. |
| Database | **PostgreSQL 16** | Open-source relational database. |
| Schema migrations | **Flyway** | Versioned SQL migrations applied at startup: every installation has the same schema. |
| API documentation | **springdoc-openapi** | Generates the OpenAPI document and Swagger UI from the controllers. |
| Frontend language | **TypeScript** | Types shared between the API client and the pages catch mistakes at compile time. |
| Frontend library | **React 19** | Component model for the pages; a small app, so no router or state-management library is needed. |
| Frontend build | **Vite** | Fast development server (with a proxy to the API) and optimized production build. |
| Containers | **Docker**, **Docker Compose** | One command starts database, backend and frontend on any operating system. |
| Web server | **nginx** | Serves the built frontend and forwards `/api` to the backend, so the browser sees a single origin. |
| Backend build | **Maven** | Standard Java build |
| CI/CD | **GitHub Actions**, **semantic-release** | Checks every change; releases versions and images from the commit history. |

### Libraries

**Backend**

| Library | Purpose |
|---|---|
| Spring Boot starters (Web MVC, Data JPA, Flyway) | HTTP layer, database access, migrations. |
| PostgreSQL JDBC driver | Connection to the database. |
| Lombok | Removes boilerplate (constructors) in Spring components and JPA entities; the domain uses Java records instead. |
| springdoc-openapi | OpenAPI document and Swagger UI. |
| JUnit 5, AssertJ | Unit tests. |
| Testcontainers | Integration tests against a real PostgreSQL in a container. |
| ArchUnit | Tests that check the architecture rules (framework-free core). |
| JaCoCo | Test coverage. |
| Spotless (Palantir Java Format) | Automatic, uniform formatting. |
| SpotBugs | Static analysis for common bugs. |

**Frontend**

| Library | Purpose |
|---|---|
| React, React DOM | User interface. |
| @fontsource/neuton | The typeface, bundled with the app instead of loaded from an online font service. |
| Vitest, Testing Library, jsdom | Unit and component tests. |
| ESLint, typescript-eslint, Prettier | Static analysis and formatting. |

The runtime dependencies of the frontend are deliberately few (React and a font): routing,
data loading and translations are small enough to be written in the app itself
(`useLoad`, `i18n.tsx`), with no extra library to keep up to date.

### External services

At runtime the application uses **no external service**: no maps, analytics, online fonts
or cloud APIs. The tax rules, the codes of the Modello 730 and the translations are part
of the code. This is what lets the user's financial data stay on their machine (NFR-4).

External services are used only to build and distribute the software:

- **GitHub** hosts the repository and runs the CI/CD workflows (GitHub Actions);
- **GitHub Container Registry** stores the released Docker images, which users can pull
  instead of building them.
