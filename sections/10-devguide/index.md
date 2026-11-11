---
title: Developer guide
has_children: false
nav_order: 11
---

# Developer Guide

## Getting in touch

The project is developed on GitHub, at
[unibo-dtm-se-2526-helper-730/artifact](https://github.com/unibo-dtm-se-2526-helper-730/artifact).
Bugs and ideas are reported as **GitHub issues**; changes are proposed as **pull
requests** (see below).

## Setting up the environment

You need **JDK 25**, **Node 26**, **Docker** and **Git**. Any IDE works; IntelliJ IDEA and
VS Code (with the Java and ESLint/Prettier extensions) are the usual choices. Lombok is
used in the backend, so enable *annotation processing* in the IDE.

Clone the repository, then start the three parts in three terminals:

```bash
docker compose up -d db          # PostgreSQL on localhost:5432

cd backend
./mvnw spring-boot:run           # API on http://localhost:8080, Swagger UI at /swagger-ui.html

cd frontend
npm ci
npm run dev                      # app on http://localhost:5173
```

The development server of the frontend forwards `/api` to the backend, so the app works
as in production. Docker must be running also for the backend tests, which start their
own PostgreSQL.

## Running the checks

Before opening a pull request, run what the CI runs:

```bash
# backend/
./mvnw verify                          # all tests, coverage in target/site/jacoco
./mvnw spotless:apply                  # fix the formatting
./mvnw spotbugs:check                  # static analysis

# frontend/
npm test                               # tests (npm run coverage for the report)
npm run typecheck && npm run lint
npm run format                         # fix the formatting
```

If you change the API, regenerate its OpenAPI document, otherwise a test fails:
`./mvnw verify "-Dopenapi.update=true"`, then commit `docs/openapi.json`.

## Code conventions

- **Formatting is automatic**: Palantir Java Format (via Spotless) for Java, Prettier for
  TypeScript. Don't format by hand: run the commands above.
- **The tax core stays framework-free**: code in `domain` and `engine` must not use
  Spring, JPA or JSON libraries. A test (`ArchitectureTest`) fails if it does.
- **Business rules live in the domain**: validate data in the constructors of the domain
  records, not in the controllers.
- **Money is `BigDecimal`**, never `double`, and is rounded with the helpers in `Money`.
- **Every text for the user is a message key**, in both
  `backend/src/main/resources/i18n/messages*.properties` (or in `frontend/src/i18n.tsx`),
  in Italian and English. A test checks that both languages have the same keys.
- **Names and comments are in English**; tests describe the behaviour in their name
  (e.g. `lossesToCarryForwardGoInT102InTheColumnOfTheirYear`).

## Workflow

1. Update `main` and create a branch named after the kind of change:
   ```bash
   git switch main && git pull
   git switch -c feat/my-change     # or fix/…, test/…, docs/…, build/…, ci/…
   ```
2. Commit with **Conventional Commits** messages, e.g.
   `fix(engine): use the oldest loss first`. Keep code and tests in separate commits
   when it helps the review.
3. Push and open a pull request to `main`. Its **title** must be a Conventional Commit
   too, because pull requests are squash-merged and the title becomes the commit on `main`
   (and decides the next version, see [Release](../06-release)).
4. Merge when the CI is green. The release, if any, happens automatically.

## Common extensions

**A new broker.**
1. Write a class implementing `BrokerParser` in `ingestion`, with the header it recognises,
   the mapping of each row and the broker's country.
2. Register it in `CoreConfig`.
3. Add its display name in `frontend/src/pages/ImportPage.tsx`, and the broker to the
   `error.unsupported_broker` message.
4. If its country is new, add its state code in `reporting/Mappings`.
5. Add a sample export under `backend/src/test/resources/golden/` and a golden test like
   `RevolutGoldenTest`.

**New tax rules** (e.g. a new rate from a given year): implement `TaxRuleSet` and select
it by year in `RuleSetFactory`.

**A new version of the Modello 730** (rows or codes moved): add a `Layout` in
`reporting/Mappings` for the years it applies to.