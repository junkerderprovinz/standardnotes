# Configuration Reference

Every variable the Unraid template exposes, with the same defaults as the
upstream [`.env.sample`](https://github.com/standardnotes/server/blob/main/.env.sample).

## Database

| Variable | Default | Notes |
|---|---|---|
| `DB_HOST` | *required* | IP address of your MariaDB container, e.g. `192.168.x.x`. |
| `DB_PORT` | `3306` | |
| `DB_USERNAME` | `std_notes_user` | |
| `DB_PASSWORD` | *required* | |
| `DB_DATABASE` | `standard_notes_db` | Must already exist; empty is fine. |
| `DB_TYPE` | `mysql` | **Internal driver value required for MariaDB.** Standard Notes / TypeORM uses the `mysql` driver string to talk to MariaDB. Do not change. |

### Provisioning the database

```sql
CREATE DATABASE IF NOT EXISTS standard_notes_db
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'std_notes_user'@'%' IDENTIFIED BY 'use-a-strong-password-here';
GRANT ALL PRIVILEGES ON standard_notes_db.* TO 'std_notes_user'@'%';
FLUSH PRIVILEGES;
```

Schema migrations run automatically on the first start of the
Standard Notes container.

## Cache

| Variable | Default | Notes |
|---|---|---|
| `REDIS_HOST` | *required* | IP address of your Redis container, e.g. `192.168.x.x`. |
| `REDIS_PORT` | `6379` | |
| `CACHE_TYPE` | `redis` | Leave at this value. |

A vanilla `redis:7-alpine` container with no auth is sufficient.

> 🔒 **Redis security.** This template does not expose a Redis
> password field. The official `standardnotes/server` image does not
> document a Redis-auth env var. Keep Redis isolated on a trusted
> VLAN / private bridge and firewall it off the public internet. Only
> add an auth env var if your specific image fork documents it.

## Secrets

All three are required and must be set to a 32-byte hex string. Generate
with `openssl rand -hex 32` (one invocation per variable).

| Variable | What it does | If you lose it |
|---|---|---|
| `AUTH_JWT_SECRET` | Signs auth tokens. | All sessions invalidated; users sign in again. |
| `AUTH_SERVER_ENCRYPTION_SERVER_KEY` | Encrypts data at rest, server-side. | **Permanent data loss** — back this up. |
| `VALET_TOKEN_SECRET` | Signs short-lived upload/download tokens for the files server. | New uploads/downloads fail until rotated; existing data unaffected. |

## Ports

| Container Port | Host Port | Purpose |
|---|---|---|
| `3000/tcp` | `3000` | API gateway. Put behind a reverse proxy. |
| `3104/tcp` | `3125` | Files server. Per upstream defaults. |

## Volumes

| Container Path | Suggested Host Path | Purpose |
|---|---|---|
| `/var/lib/server/logs` | `/mnt/user/appdata/standardnotes/logs` | Server logs. |
| `/opt/server/packages/files/dist/uploads` | `/mnt/user/appdata/standardnotes/uploads` | Encrypted file uploads. |

## Optional

| Variable | Default | Notes |
|---|---|---|
| `PUBLIC_FILES_SERVER_URL` | *(empty)* | Set to the public HTTPS URL of the files server when reverse-proxying it on a different hostname. |

For anything beyond this — disabling user registration, custom SMTP,
extension server, payments — refer to the upstream
[`.env.sample`](https://github.com/standardnotes/server/blob/main/.env.sample)
and [self-hosting docs](https://standardnotes.com/help/self-hosting/getting-started).
Add the corresponding variables as **Variable** entries to the Unraid
template (Add another Path, Port, Variable, Label or Device).
