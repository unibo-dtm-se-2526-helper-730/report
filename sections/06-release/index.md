---
title: Release
has_children: false
nav_order: 7
---

# Release

## Artefacts

Each release produces **two Docker images**, plus a tag and a release page on GitHub:

| Artefact | Content | Repository |
|---|---|---|
| `helper-730-backend` image | The Spring Boot application (`app.jar`) on a Java 25 runtime (`eclipse-temurin:25-jre`). | GitHub Container Registry (`ghcr.io/unibo-dtm-se-2526-helper-730/helper-730-backend`) |
| `helper-730-frontend` image | The built single-page app served by nginx, with the proxy to the backend. | GitHub Container Registry (`ghcr.io/unibo-dtm-se-2526-helper-730/helper-730-frontend`) |
| Git tag `vX.Y.Z` and **GitHub Release** | The source code at that version and the release notes, generated from the commits. | GitHub |

The database is not an artefact of the project: the official `postgres:16-alpine` image is
used as it is. No JAR or npm package is published, because the application is meant to
be run as a whole, with Docker Compose, not used as a library.

**Why GitHub Container Registry.** It lives next to the code: images are linked to the
repository (through the `org.opencontainers.image.source` label), are free for public
repositories, and are pushed by the CI with the token GitHub already provides, so no
extra account or secret is needed (unlike Docker Hub).

## How artefacts are released

Releases are **fully automatic**, done by [semantic-release](https://semantic-release.gitbook.io/)
in the `deploy` workflow (see [CI/CD](../08-cicd)). After every merge into `main`, once all
the checks and tests have passed:

1. semantic-release reads the commits since the last tag and computes the next version
   (see below); if there is nothing to release, it stops here;
2. the script `.github/scripts/publish-images.sh` writes the version into the backend
   build, then builds both images and pushes them to GHCR, each with three tags:
   `X.Y.Z`, `X.Y` and `latest`;
3. only if the images were pushed, semantic-release creates the tag `vX.Y.Z` and the
   GitHub Release with the notes.

If building or pushing an image fails, no tag and no release are created, so a version
number never points to missing images. The version is never committed to the code
(`pom.xml` and `package.json` keep a placeholder): the Git tag is the only source of
truth.

On any branch other than `main` the same job runs as a **dry run**: it prints the version
that branch would produce once merged, and publishes nothing.

**Configuration.** Nothing has to be set up by hand: the workflow uses the
`GITHUB_TOKEN` that GitHub gives to every run, with permission to write packages and
contents. An optional `RELEASE_TOKEN` secret can replace it. The configuration is in
`release.config.js` (versioning rules and plugins) and `.github/workflows/deploy.yml`.
A release can also be started by hand from **Actions → deploy → Run workflow**.

**Using a released version.** The published images can be pulled directly, by version or
as `latest`:

```bash
docker pull ghcr.io/unibo-dtm-se-2526-helper-730/helper-730-backend:0.6.2
docker pull ghcr.io/unibo-dtm-se-2526-helper-730/helper-730-frontend:0.6.2
```

## Choice of the license

Both the code and the artefacts are released under the **Apache License 2.0**.

- It is **permissive**: anyone can use, modify and redistribute the software, also in
  commercial products, as long as they keep the copyright and license notices. This fits
  a tool that other developers might extend with new brokers or tax years.
- Unlike MIT, it includes an explicit **patent grant** and states clearly how
  contributions are licensed.
- It is **compatible with the dependencies**: Spring Boot is Apache 2.0, React and most
  frontend libraries are MIT.

The same license covers code and images because the images contain only the project's
own build output on top of third-party base images (nginx, Eclipse Temurin), which keep
their own licenses.

## Choice of the versioning schema

Releases follow **[Semantic Versioning](https://semver.org/)** (`MAJOR.MINOR.PATCH`), with
the version computed from **Conventional Commits** (see [Development](../04-development)):

| Commits since the last release | Next version | Example |
|---|---|---|
| at least one `fix:` or `perf:` | patch | 0.6.1 → 0.6.2 |
| at least one `feat:` | minor | 0.5.3 → 0.6.0 |
| a breaking change (`feat!:` or `BREAKING CHANGE:` in the body) | major | 0.6.2 → 1.0.0 |
| only `docs:`, `test:`, `ci:`, `build:`, `chore:`, `style:` | no release | — |

**Why SemVer.** The version tells users what to expect from an update: a patch can be
taken without thinking, a minor adds features, a major may need them to change something
(for example a new database migration that cannot be undone). Deriving it from the
commits removes the human decision, and with it the mistakes.

The versions are still **0.x**: the project is in initial development, and the API and
data format may change. Version 1.0.0 will be released with a deliberate breaking-change
commit, once the supported features are stable.

**Aligned versions.** Backend and frontend always have the **same version**, because they
are released together by the same job from the same commit. The frontend is built for one
version of the API, so separate version numbers would only force users to find out which
pairs work together.

**How a new version is created.**

- *When:* at every merge into `main` that contains at least a `fix` or a `feat`; nobody
  decides the number by hand.
- *Release branches:* not used. `main` is always releasable, and every change reaches it
  through a short-lived branch and a pull request.
- *Tags:* created by semantic-release, as `vX.Y.Z` on the merge commit; they are never
  created by hand.
- *GitHub Releases:* created by semantic-release together with the tag, with release
  notes grouped by type (features, bug fixes) and links to the pull requests.

So far the project has been released from **v0.1.0** to **v0.6.2**.