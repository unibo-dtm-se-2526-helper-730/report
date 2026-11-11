---
title: Validation
has_children: false
nav_order: 6
---

# Validation

## Testing approach

Testing follows the **test pyramid**: many fast unit tests on the tax core, fewer
integration tests on the whole backend with a real database, and a manual acceptance
pass through the user interface.

- **Focus on the tax core.** The domain and engine contain the rules that decide how much
  the user pays, so they have the most tests, and every rule is checked with amounts
  computed by hand. Since the core is framework-free, these tests are plain Java, fast
  and need no database or server.
- **Reference files.** Two sample broker exports (one for Revolut, one for Trade
  Republic) are used as *golden files*: their expected results were computed by hand once,
  and the tests check that the application keeps producing them.
- **Tests written with the code.** Each feature was delivered together with its tests in
  the same pull request (usually as a separate `test(...)` commit). Strict TDD was not
  followed.
- **Automation.** All automated tests run in the CI pipeline at every push and pull
  request (see [CI/CD](../08-cicd)); a change is merged only when they pass.

| Part | Frameworks | Why |
|---|---|---|
| Backend | **JUnit 5** and **AssertJ** | Standard for Java; parameterized tests cover all asset classes with one test; AssertJ compares `BigDecimal` by value. |
| | **Spring Boot Test** with **MockMvc** | Runs the whole application in the test and sends real HTTP requests to it, without starting a server. |
| | **Testcontainers** | Starts a real PostgreSQL 16 in Docker for the integration tests. |
| | **ArchUnit** | Turns the architecture rules into tests. |
| | **JaCoCo** | Coverage of unit and integration tests together. |
| Frontend | **Vitest**, **Testing Library**, jsdom | Native to Vite; Testing Library tests the pages as the user sees them (texts, roles, buttons). |

## Testing (automated)

### Unit testing

| Area | Test classes | What they check | Requirements |
|---|---|---|---|
| Domain | `TransactionTest`, `InstrumentTest`, `CapitalLossTest` | Validation of every field (ISIN check digit, rates between 0 and 1, positive amounts), tax year in the Italian time zone, implied fees. | FR-3, FR-4 |
| Lots | `LotBookTest`, `WeightedAverageCostTest` | LIFO with fees, weighted average without fees, partial sales, costs of partial sales adding up exactly to the lot cost, sales larger than the holding. | FR-6, FR-7, FR-13, NFR-1 |
| Tax rules | `CurrentRegimeTest`, `ClassifierTest` | Income category and taxable share for every asset class (48.08% for government bonds, share of government bonds of ETFs), gross-up of dividends, unclassified instruments. | FR-5, FR-8, FR-9 |
| Zainetto | `ZainettoTest` | Netting within the year, use of losses oldest first, expiry after four years, losses entered by hand, capital income never offset. | FR-11, FR-12 |
| Tax of a year | `TaxCalculatorTest` | Taxes by category, payment threshold of 12 euro, unsupported years, unclassified instruments listed all at once. | FR-5, FR-10 |
| IVAFE | `IvafeCalculatorTest` | Days of holding, value at the sale or at 31 December, missing prices. | FR-14 |
| Declaration | `DeclarationBuilderTest` | Values of T11, T102, M31 and W, split between 26% and 12.5%, warnings. | FR-15 |
| Portfolio | `PortfolioCalculatorTest` | Holdings at cost by instrument and asset class. | FR-16 |
| Ingestion | `CsvReaderTest`, `RevolutParserTest`, `TradeRepublicParserTest`, `ValidatorsTest` | Number formats, every operation type, faulty rows reported without stopping the import, corporate actions, non-EUR currencies, future dates, high implied fees. | FR-1, FR-3 |
| Translations | `MessagesTest` | Italian and English texts, number formats per language, same keys in both languages. | FR-18 |
| Architecture | `ArchitectureTest` | Domain and engine do not depend on frameworks; the domain depends on no other layer. | NFR-5 |
| Frontend | `api.test`, `format.test`, `ImportPage.test`, `DeclarationPage.test` | API client (URLs, methods, language header, errors), formatting of amounts and dates per language, import and deletion flow, warnings of the declaration. | FR-1, FR-17, FR-18 |

**Results.** All tests pass. On the last full run there were **150 backend unit tests**
and **29 frontend tests**.

### Integration testing

`ApiIntegrationIT` tests **all the backend components together**: REST controllers,
services, parsers, tax engine, persistence and a real **PostgreSQL** database. Requests
go through the HTTP layer with MockMvc, and the results are checked in the JSON
responses:

| Test | Rationale | Requirements |
|---|---|---|
| Golden Revolut CSV through the whole application | Import, classification, tax of the year: the same values as the golden file. | FR-1, FR-10 |
| Golden Trade Republic CSV | Same, plus Germany as the state of quadro W. | FR-1, FR-15 |
| Re-importing the same file | No new transaction, all rows reported as duplicates. | FR-2 |
| Tax with unclassified instruments | Answer 409 with the list of tickers. | FR-5 |
| Invalid classification | Answer 400 with a translated message. | FR-4 |
| Replacing the prices of a year | One price per ticker and year. | FR-14 |
| Tax year in Italian time | An operation at 23:30 UTC on 31 December belongs to the next year. | FR-10 |
| Delete all, with and without instruments | Transactions, history, prices and unused instruments removed as expected. | FR-17 |

`OpenApiSpecIT` checks that the API matches the OpenAPI document committed in the
repository, so the documentation cannot drift from the code.

**Results.** 10 integration tests, all passing. Coverage of unit and integration tests
together (JaCoCo):

| Package | Instructions | Branches |
|---|---|---|
| `engine` | 98% | 90% |
| `engine.ivafe`, `engine.zainetto` | 100% | 100% |
| `engine.rules` | 93% | 81% |
| `domain` | 97% | 85% |
| `domain.zainetto` | 93% | 62% |
| `reporting` | 98% | 94% |
| `ingestion` | 97% | 88% |
| `ingestion.revolut` | 90% | 93% |
| `ingestion.traderepublic` | 85% | 85% |
| `ingestion.validation` | 100% | 100% |
| `api` | 79% | 58% |
| `service` | 74% | 39% |
| `i18n` | 63% | 45% |
| **Whole backend** | **92%** | **83%** |

Frontend coverage is lower (**35% of statements, 18% of branches**): the tests cover the
API client, formatting and the import and declaration pages, while the other pages are
checked manually (see below).

**Test doubles.**

- *Fake clock*: parsers and validators receive a fixed `Clock`, so that "future date"
  checks do not depend on the day the tests run.
- *Mocks and stubs in the frontend*: `fetch` is replaced by a mock that returns a given
  response and records the request, to check the API client; pages use a stubbed API
  module that returns fixed data.
- *No doubles for the database*: integration tests use a real PostgreSQL rather than an
  in-memory database or mocked repositories, so that SQL, constraints and migrations are
  tested as in production.

### System testing

The integration tests above are also the **system tests of the backend**:
- they run the whole application, as configured in production, on a real database;
- they use the same sample files a user would import;
- they check the acceptance criteria of FR-1, FR-2, FR-5, FR-10, FR-15 and FR-17.

There is no automated end-to-end test through the browser: the user interface is covered
by its component tests and by the manual acceptance tests.

**Containers.**

- *Database*: Testcontainers starts a fresh `postgres:16-alpine` container for each run,
  so every run starts from an empty database with the migrations applied, and nothing
  depends on a database installed on the developer's machine.
- *CI*: the pipeline runs all tests on Linux, and the unit tests also on Windows and
  macOS, with Java 25 and 26. It also builds the Docker images of backend and frontend,
  to check that the system can be packaged.

## Acceptance tests (manual)

The acceptance criteria of the requirements were checked by hand on the application
started with `docker compose up`, in a clean installation (`docker compose down -v`
first), with the two sample files in `backend/src/test/resources/golden/`.

| # | Steps | Expected result | Requirement |
|---|---|---|---|
| 1 | Import `revolut_golden.csv`. | Report with the imported rows; the years appear in the menu. | FR-1 |
| 2 | Import the same file again. | All rows reported as duplicates; nothing changes. | FR-2 |
| 3 | Import a CSV of another broker or a text file. | Error naming the supported brokers. | FR-1 |
| 4 | Open the summary of 2026 before classifying. | Warning listing the instruments to classify, with a link to the page. | FR-5 |
| 5 | Classify 5J50 as share and the others as UCITS ETF; open the summary of 2026. | Taxes by category; the 5J50 sale expands into its lot; a loss of 16.90 in the Zainetto. | FR-6, FR-8, FR-11, FR-13 |
| 6 | Enter a loss of 30 euro for 2024. | It appears in the Zainetto of 2026 with its expiry; no warning. | FR-12 |
| 7 | Enter a loss for 2026, a year already imported. | Warning about counting it twice; saving is still possible. | FR-12 |
| 8 | Enter the prices at 31/12/2026 and open IVAFE. | One row per holding period, with days and tax. | FR-14 |
| 9 | Open the declaration of 2026. | Values in T11, T102, M31 and W, with the codes of the other columns. | FR-15 |
| 10 | Open the portfolio. | Holdings at cost and split by asset class. | FR-16 |
| 11 | Switch the language to English on every page. | All labels, warnings and errors in English. | FR-18 |
| 12 | Delete all transactions, first cancelling, then confirming. | Cancel: nothing changes; confirm: no transaction or import is left. | FR-17 |

**Results.** 12 of 12 passed.