---
title: Deployment
has_children: false
nav_order: 8
---

# Deployment

Helper Dichiarazione 730 runs entirely on the user's computer: there is no server to set
up and no online service to sign up for.

## User installation

The user needs only **Docker**, to run the three containers (database, backend, frontend),
and a **web browser**. Java, Node.js and PostgreSQL do *not* have to be installed: they
are inside the containers.

### 1. Install Docker

- **Windows and macOS:** install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
  and start it.
- **Linux:** install Docker Engine and the Compose plugin, e.g. on Ubuntu:

  ```bash
  sudo apt install docker.io docker-compose-v2
  ```

Check that it works:

```bash
docker compose version
```

### 2. Get the application

Clone the repository, or download it as a ZIP from GitHub (*Code → Download ZIP*) and
extract it:

```bash
git clone https://github.com/unibo-dtm-se-2526-helper-730/artifact.git helper-730
cd helper-730
```

### 3. Start it

```bash
docker compose up --build -d
```

The first start builds the images and takes a few minutes; the next ones take seconds.
Docker Compose starts the database first and the backend only when the database is ready;
the backend then creates the database tables by itself (Flyway migrations).

Open **<http://localhost:3000>** in the browser.

### Configuration (optional)

No configuration is needed: the defaults work out of the box. The database name, user and
password default to `helper730` and can be changed with environment variables, or with a
`.env` file next to `docker-compose.yml`:

```bash
# .env
DB_NAME=helper730
DB_USER=helper730
DB_PASSWORD=choose-a-password
```

The application uses ports **3000** (application), **8080** (API) and **5432**
(database): they must be free on the computer. Since the ports are published on the
computer's network interfaces, the application should be used on a trusted network, as it
has no login (see [Development](../04-development)).

### Everyday use

| Action | Command |
|---|---|
| Stop the application (data is kept) | `docker compose down` |
| Start it again | `docker compose up -d` |
| Update to the latest version | `git pull`, then `docker compose up --build -d` |
| Remove the application **and all its data** | `docker compose down -v` |

The data (transactions, classifications, prices, losses) is kept in the Docker volume
`pgdata`, so it survives restarts and updates; only `down -v` deletes it.

Instead of building the images, the released ones can be pulled from GitHub Container
Registry (see [Release](../06-release)).

## Server-side installation

**No server is needed.** The backend and the database, which in a typical web application
would run on a server, run here as containers on the user's own computer, started by
Docker Compose together with the frontend.

This is a deliberate choice (see [Design](../03-design)): the application handles the
user's financial data, and keeping it on their machine means there is no server to
secure, no user accounts, and no data sent to third parties.

**Further software.** The only other software the system needs is the **PostgreSQL 16**
database, and it is installed and configured automatically: Docker Compose downloads the
official `postgres:16-alpine` image, creates the database with the configured name, user
and password, and stores its files in the `pgdata` volume. Nothing has to be installed or
configured by hand.