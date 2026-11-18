# Gitea

[Gitea](https://gitea.io/) is a self hosted alternative to using
github. It provides a GUI for interacting with git repos and provides
some of the functionality which is available within github.

The benefit of self hosting means that you ensure that repos are protected and
kept entirely private rather than hosting on the web.

## Docker Compose Setup

The stack in [compose.yaml](/Users/mikehorrocks/Projects/docker/gitea/compose.yaml)
runs two containers on a dedicated bridge network:

| Service  | Container | Image             | Purpose                              |
|----------|-----------|-------------------|--------------------------------------|
| `server` | `gitea`   | `gitea/gitea:28-nightly-rootless` | Gitea web UI, Git over HTTP, and SSH |
| `db`     | (auto)    | `mysql:8.4`       | MySQL database backing Gitea         |

### Ports

| Host port | Container port | Use                    |
|-----------|----------------|------------------------|
| 3000      | 3000           | Web UI / HTTP Git      |
| 2222      | 2222           | SSH Git (`ssh://git@localhost:2222/...`) |

MySQL is not published to the host; it is only reachable from the `gitea`
network as `db:3306`.

### Networking

The `gitea` network uses the subnet `192.168.230.0/24`. Both services join it,
so Gitea reaches the database by the service name `db`.

### Volumes

Everything is a bind mount relative to the project directory:

| Host path        | Container path        | Contents                                      |
|------------------|-----------------------|-----------------------------------------------|
| `./gitea-data`   | `/var/lib/gitea`      | Repositories, attachments, avatars, sessions, SSH host keys, JWT key, indexes |
| `./gitea-config` | `/etc/gitea`          | `app.ini`                                     |
| `./mysql-data`   | `/var/lib/mysql`      | MySQL data files                              |
| `/etc/timezone`, `/etc/localtime` | same (read-only) | Host timezone                |

### Gitea configuration

Gitea reads [app.ini](/Users/mikehorrocks/Projects/docker/gitea/gitea-config/app.ini).
Notable settings:

- `[server]`: HTTP on port `3000`, SSH server enabled on port `2222`, `SSH_DOMAIN = localhost`, `ROOT_URL` left blank so it is derived from the request.
- `[database]`: MySQL at `db:3306`, database and user `gitea`.
- `[security]`: `INSTALL_LOCK = true`, so the web installer is skipped.
- `[service]`: `DISABLE_REGISTRATION = false`, so anyone who can reach the UI can create an account.

Environment variables in `compose.yaml` in the form `GITEA__<section>__<KEY>`
override the matching `app.ini` values. The database settings are set in both
places and currently agree.

The `gitea` container runs the rootless image as UID/GID `1000`
(`USER_UID`/`USER_GID`). The host folders `gitea-data` and `gitea-config` must
be writable by that user.

### Database

`db` uses `mysql:8.4` with the database and user `gitea`. The root and user
passwords come from `.env` (see `MYSQL_*` in `compose.yaml`). It has a healthcheck
(`mysqladmin ping`, every 10s), and the `gitea` service waits for it to report
healthy before starting.

### Configuration secrets

Passwords are read from `.env`, which Docker Compose loads automatically. It is
git-ignored. To set up a new environment:

```bash
cp .env.example .env
# edit .env and replace every change-me-* value
```

Compose refuses to start if a required variable is missing from `.env`.

### Running

```bash
# Start in the background
docker compose up -d

# Follow Gitea logs
docker compose logs -f server

# Stop the stack (data is kept in the bind-mounted folders)
docker compose down
```

Once running, open http://localhost:3000. Clone over SSH with
`git clone ssh://git@localhost:2222/<owner>/<repo>.git`.

### Security notes

- Keep `.env` out of version control (already git-ignored). Use strong,
  unique passwords. If you change `MYSQL_PASSWORD` or `GITEA_DB_PASSWORD`
  after the database is initialised, the values in `.env` must be updated
  to match the database itself.
- `gitea-data/` holds SSH host keys and the JWT private key, and `mysql-data/`
  holds MySQL key and certificate files. Keep these out of version control.
- Port 3000 and 2222 are published on all host interfaces.


