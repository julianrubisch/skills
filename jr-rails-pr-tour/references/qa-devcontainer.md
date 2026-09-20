# Ephemeral QA devcontainer

For a project without `.devcontainer/`, the tour can still run the PR in a
container without writing to the repository: generate a devcontainer in the
scratchpad and pass it with `--config`. Nothing is committed, nothing is
proposed to the project; the files live and die with the session.

Offer it once, as one question, only after the tour is published: "Run the
PR in a throwaway container for QA, or on your host with `bin/dev`?" Default
to the host command in `qa.start` either way; on a yes, do the steps
below and update `qa.environment`, `qa.start` and `qa.base_url`.

## Detect what the app needs

| Signal | Read from | Use |
|---|---|---|
| Ruby version | `.ruby-version`, `.tool-versions`, `Gemfile` `ruby "…"` | image tag |
| Database | `Gemfile`: `pg` → postgres, `mysql2`/`trilogy` → mysql, `sqlite3` → none | service + `DATABASE_URL` |
| Redis | `Gemfile`: `redis`, `sidekiq`, `hiredis` | service + `REDIS_URL` |
| Node | `package.json` present | `node` feature |
| Web command | `Procfile.dev` `web:` line, else `bin/rails server -b 0.0.0.0` | start command |

The app name for the database URL is the `database:` value in
`config/database.yml` under `development`, or the directory name.

## Files to write into `$SCRATCH/qa-devcontainer/`

`devcontainer.json`:

```json
{
  "name": "pr-tour-qa",
  "dockerComposeFile": "compose.yml",
  "service": "app",
  "workspaceFolder": "/workspace",
  "features": {
    "ghcr.io/devcontainers/features/node:1": {}
  },
  "containerEnv": {
    "DATABASE_URL": "postgres://postgres:postgres@postgres/APP_development",
    "REDIS_URL": "redis://redis:6379/0",
    "RAILS_ENV": "development"
  },
  "postCreateCommand": "bin/setup --skip-server || (bundle install && bin/rails db:prepare)"
}
```

`compose.yml`:

```yaml
services:
  app:
    image: ghcr.io/rails/devcontainer/images/ruby:RUBY_VERSION
    volumes:
      - REPO_ABSOLUTE_PATH:/workspace:cached
    command: sleep infinity
    ports:
      - "3000:3000"
    depends_on: [postgres, redis]
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - pg:/var/lib/postgresql/data
  redis:
    image: redis:7
volumes:
  pg:
```

Adjust to the detection table: drop `redis` when absent (and its
`depends_on` and `REDIS_URL`); swap `postgres` for
`mysql:8` with `MYSQL_ROOT_PASSWORD` and
`DATABASE_URL=mysql2://root:root@mysql/APP_development` for MySQL; drop
the database service and `DATABASE_URL` entirely for SQLite. Drop the
`node` feature when there is no `package.json`. Replace `RUBY_VERSION` with
the detected version (`3.4.1`, not `ruby-3.4.1`); if the image tag does
not exist, fall back to `ruby:RUBY_VERSION` from Docker Hub with
`build-essential libpq-dev` added via a `Dockerfile`.

`ports: 3000:3000` in compose is what publishes the app on the host;
`forwardPorts` would not, under the CLI. `DATABASE_URL` overrides
`config/database.yml`, so the project's file needs no change.

`REPO_ABSOLUTE_PATH` is the checkout's absolute path. A relative path
would resolve against the scratchpad, where `compose.yml` lives, not
against the repository.

## Run

```bash
CFG="$SCRATCH/qa-devcontainer/devcontainer.json"
devcontainer up   --workspace-folder "$REPO" --config "$CFG"
devcontainer exec --workspace-folder "$REPO" --config "$CFG" bin/dev   # background
```

Set `qa.environment` to `devcontainer`, `qa.base_url` to
`http://localhost:3000`, and `qa.start` to the two commands above so the
reviewer can restart it. Mention in the tour lede that the container is
throwaway and lives in the session's scratchpad.

## Stop

Tell the reviewer, do not run it unasked:

```bash
docker compose -f "$SCRATCH/qa-devcontainer/compose.yml" down -v
```

`-v` removes the database volume; without it the next PR's tour reuses
stale data.
