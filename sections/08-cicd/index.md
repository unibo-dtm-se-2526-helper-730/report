
| Stage | What is automated | Why |
|---|---|---|
| **check** | Compilation, static analysis, formatting, all tests with coverage, build of the frontend and of the Docker images. | Catch every kind of problem on every change, before it reaches `main`. |
| **test** | The backend tests on Linux, Windows and macOS, with Java 25 and 26. | The application must run on the users' systems (NFR-7), and on the next Java version. |
| **deploy** | Version number, Docker images, tag and GitHub Release (see [Release](../06-release)). | Releasing by hand is slow and error-prone: here it is a consequence of merging. |

Each stage runs only if the previous one succeeded, so nothing is released unless every
check and test is green. **Nothing is done by hand**: a developer only opens a pull
request and merges it.

- **Continuous integration.** The pipeline runs at every push, on every branch, and on
  every pull request. `main` changes only through pull requests, and a pull request is
  merged only when its pipeline is green, so `main` is always buildable and tested.
- **Continuous delivery.** Every merge into `main` that contains a `feat` or a `fix` is
  released automatically: the new images are on GitHub Container Registry a few minutes
  after the merge. Deployment on the users' machines remains their choice (they pull or
  rebuild the new version), since the application runs locally.

## Implementation with GitHub Actions

The pipeline is split into two workflow files:

- `.github/workflows/check.yml` (named **CI/CD**) contains the `check` and `test` jobs
  and calls the deploy workflow;
- `.github/workflows/deploy.yml` contains the release. It is a *reusable* workflow
  (`workflow_call`): it has no triggers of its own, so it can run only after the checks,
  or by hand.

### Triggers

- **push** on any branch, except bots' branches (`dependabot/**`, `renovate/**`) and
  changes to files that cannot break anything (`.gitignore`, `LICENSE`, `README.md`);
- **pull_request**: the PR is checked and tested, but the `deploy` job is skipped
  (`if: github.event_name != 'pull_request'`), so a pull request can never publish
  anything;
- **workflow_dispatch**: both workflows can be started by hand from the *Actions* tab.

### Jobs

**check** (Ubuntu), in order, failing at the first error:

| Step | Command |
|---|---|
| Compile backend and tests | `./mvnw test-compile` |
| Type-check frontend | `npm run typecheck` |
| Static analysis | `./mvnw spotbugs:check`, `npm run lint` |
| Formatting | `./mvnw spotless:check`, `npm run format:check` |
| Backend tests and coverage | `./mvnw verify` (unit tests, integration tests on PostgreSQL with Testcontainers, JaCoCo report) |
| Frontend tests and coverage | `npm run coverage` |
| Builds | `npm run build`, `docker build` of both images (built, not published) |

The cheap checks come first, so a formatting mistake fails in seconds instead of after
the tests. The coverage reports are uploaded as **artifacts** of the run
(`coverage-report-<commit>` and `coverage-report-frontend-<commit>`), downloadable from the
run page.

**test**: a **matrix** of 3 operating systems × 2 Java versions = 6 jobs, run in parallel
with `fail-fast: false`, so one failure does not hide the others. Linux runs all the
tests (`verify`); Windows and macOS run the unit tests only (`test`), because the
integration tests need Linux containers, which only the Linux runners provide. Coverage
is skipped here (`-Djacoco.skip=true`), since it is already measured in `check`.

**deploy**: runs `semantic-release` (see [Release](../06-release)). It checks out the
whole history (`fetch-depth: 0`), because the next version depends on the commits since
the last tag, sets up Docker Buildx, logs in to GHCR, and publishes.

### Secrets, permissions and environment variables

| Name | Kind | Use |
|---|---|---|
| `GITHUB_TOKEN` | Secret, provided by GitHub at every run | Logs in to GHCR to push the images; creates the tag and the GitHub Release. |
| `RELEASE_TOKEN` | Optional repository secret | Replaces `GITHUB_TOKEN` for the release, if a personal token is needed. Not required. |
| `RELEASE_DRY_RUN` | Environment variable | `true` on every branch except `main`: semantic-release only prints the next version. |
| `GITHUB_REPOSITORY_OWNER`, `GITHUB_SHA` | Variables provided by GitHub | Name of the images (`ghcr.io/<owner>/…`) and commit label added to them. |

- **Least privilege.** The CI/CD workflow runs with read-only access to the repository
  (`permissions: contents: read`). Only the `deploy` job gets write permissions: contents
  for the tag and release, packages for the images, issues and pull requests for the
  "released in" comments. The secrets are passed to it explicitly (`secrets: inherit`).
- **No stored credentials.** The checkout of the release job does not keep the token in
  the local Git configuration (`persist-credentials: false`); semantic-release uses its own.

### Other details

- **Caching.** Maven and npm dependencies are cached between runs (`setup-java` and
  `setup-node` with `cache`), which saves minutes on every run.
- **Concurrency.** A new push to a branch cancels the run still in progress on that branch
  (`cancel-in-progress`), to save time and runners, except on `main`, where a run might be
  in the middle of a release. Releases themselves never overlap (`concurrency: deploy`).
- **Same versions as development.** The jobs use the Java (25) and Node (26) versions
  required by the project; the release job reads the Node version from `package.json`, so
  it cannot drift.