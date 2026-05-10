<h1 align="center">Standard Notes for Unraid</h1>

<a href="https://standardnotes.com">
  <img src=".github/assets/banner.svg" alt="Standard Notes for Unraid" width="100%">
</a>

<p align="center">
  <a href="https://hub.docker.com/r/standardnotes/server"><img src="https://img.shields.io/badge/Image-standardnotes%2Fserver-3daee9?style=for-the-badge&logo=docker&logoColor=white" alt="Image" height="36"></a>&nbsp;
  <a href="https://standardnotes.com/help/self-hosting/docker"><img src="https://img.shields.io/badge/Docs-Self--Hosting-3daee9?style=for-the-badge&logo=readthedocs&logoColor=white" alt="Docs" height="36"></a>&nbsp;
  <a href="#3-quick-start-on-unraid"><img src="https://img.shields.io/badge/Unraid-Template-f15a2c?style=for-the-badge&logo=unraid&logoColor=white" alt="Unraid" height="36"></a>&nbsp;
  <a href="#5-database--cache"><img src="https://img.shields.io/badge/Database-MariaDB-bdc3c7?style=for-the-badge&logo=mariadb&logoColor=white" alt="MariaDB" height="36"></a>&nbsp;
  <a href="#5-database--cache"><img src="https://img.shields.io/badge/Cache-Redis-bdc3c7?style=for-the-badge&logo=redis&logoColor=white" alt="Redis" height="36"></a>&nbsp;
  <a href="#7-security"><img src="https://img.shields.io/badge/E2E-encrypted-3daee9?style=for-the-badge&logo=letsencrypt&logoColor=white" alt="E2E" height="36"></a>&nbsp;
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge&logo=opensourceinitiative&logoColor=white" alt="License" height="36"></a>
</p>

<p align="center">
A clean, opinionated <b>Unraid Community Template</b> for the
<a href="https://standardnotes.com">Standard Notes</a> self-hosted backend.
Run your own end-to-end-encrypted notes server on Unraid in a few minutes,
with <b>MariaDB</b> and <b>Redis</b> as separate, reusable containers — no
bundled databases, no surprises.
</p>

<p align="center">
<i>Unofficial community wrapper. Not affiliated with or supported by Standard Notes.</i>
</p>

---

## Table of Contents

0. [⚠️ Sync-Loop / Duplicate Notes Guardrails](#0-sync-loop--duplicate-notes-guardrails)
1. [What is this?](#1-what-is-this)
2. [Architecture](#2-architecture)
3. [Quick Start on Unraid](#3-quick-start-on-unraid)
4. [Generating Secrets](#4-generating-secrets)
5. [Database & Cache](#5-database--cache)
6. [Configuration Reference](#6-configuration-reference)
7. [Security](#7-security)
8. [Reverse Proxy](#8-reverse-proxy)
9. [Backup & Restore](#9-backup--restore)
10. [Updating](#10-updating)
11. [Troubleshooting](#11-troubleshooting)
12. [Contributing / License](#12-contributing--license)

---

## 0. Sync-Loop / Duplicate Notes Guardrails

> ⚠️ **Read this before you connect a real Standard Notes account.**
> Diese Sektion bitte vor der ersten Anmeldung lesen.

Self-hosted Standard Notes installations — including this Unraid
template — can in rare configurations produce **massive note
duplication**: a single new note replicates dozens or hundreds of
times within seconds in the client. This is almost always a
**configuration** problem in the surrounding stack (sync, cookies,
proxy, cache), **not** a bug in your notes themselves. Once it starts,
every additional client write makes the situation worse.

### Common causes (Unraid / self-hosted)

- **Sync conflicts** between clients. Standard Notes' [official
  duplicate-handling docs](https://standardnotes.com/help/33/how-do-i-clear-duplicates)
  state that the **server cannot decrypt or merge note content** — when
  two clients sync conflicting versions of the same note, the app
  duplicates the conflicting copy on the client side. With several
  clients online and an unstable backend, the cascade can run away.
- **Redis unreachable or wrong host/port.** If `REDIS_HOST` /
  `REDIS_PORT` are wrong, the server logs `ECONNREFUSED` and sync
  state is not deduplicated correctly across requests.
- **Bad `COOKIE_DOMAIN` / session-cookie handling behind a reverse
  proxy.** Symptoms include `No cookies provided for cookie-based
  session token` in the server log and a `/v1/items` request loop.
  See the upstream forum
  [issue #3635](https://github.com/standardnotes/forum/issues/3635).
- **Reverse proxy serving the API over HTTP instead of HTTPS.** Modern
  clients require HTTPS; without it, `Secure` cookies are dropped and
  the session loop above can trigger.
- **IP addresses instead of Docker container names** for `DB_HOST` /
  `REDIS_HOST`. IPs change when containers restart on Unraid; container
  names on a shared custom bridge network are stable.
- **Image / client version mismatch.** The legacy
  [`standardnotes/syncing-server` issue
  #102](https://github.com/standardnotes/syncing-server/issues/102)
  documents how mixing a stable web app with a development
  syncing-server caused `Syncing: 0/1` and duplicate cascades. That
  repo is **archived and historical** — the current image is
  `standardnotes/server` — but the same class of mismatch can still
  happen if you pin to a broken `latest`.

### Hard guardrails before you migrate any real notes

1. **Test with a fresh, throwaway account first.** Do **not** point
   your existing client (with months of real notes) at this server
   until you have verified end-to-end sync with a brand-new account.
2. **Use one client at a time** during the test. Connect a second
   device only once a single note has round-tripped cleanly.
3. **Make a decrypted backup / export** of your real notes from
   `standardnotes.com` (or your existing self-hosted server) **before**
   reconnecting any existing client.
4. **If duplication starts: stop.** Stop all clients, stop the server
   container, inspect logs and database. Do **not** keep editing —
   each edit can fan out further.

Full step-by-step checklist:
[`docs/sync-loop-troubleshooting.md`](docs/sync-loop-troubleshooting.md).

### Known risk — official + historical context

- **Official:** Standard Notes' help article *How do I clear
  duplicates?* states that duplicates are an **app-side conflict
  resolution** mechanism. The server cannot decrypt note bodies, so
  it cannot merge conflicts; the client duplicates conflicting copies
  to avoid silent data loss.
  <https://standardnotes.com/help/33/how-do-i-clear-duplicates>
- **Historical / legacy:** The old archived
  [`standardnotes/syncing-server`](https://github.com/standardnotes/syncing-server)
  had a documented case
  ([issue #102](https://github.com/standardnotes/syncing-server/issues/102))
  where stable web app + `dev`/`latest` syncing-server produced
  `Syncing: 0/1` plus duplicate cascades. That repo is no longer the
  current self-host backend (the current image is
  `standardnotes/server`), but the lesson — *don't mix
  unstable image tags with stable clients* — still applies.
- **Self-hosted session loop:** Forum
  [issue #3635](https://github.com/standardnotes/forum/issues/3635)
  describes the `No cookies provided for cookie-based session token`
  symptom, traced back to `COOKIE_DOMAIN` / HTTPS / reverse-proxy
  cookie handling and an unstable image tag. Pinning a known-good
  tag was a working mitigation.

---

## 1. What is this?

This repository ships **Unraid Community Application templates** for the
[official `standardnotes/server` Docker image](https://hub.docker.com/r/standardnotes/server),
plus an optional **LocalStack** companion template that mirrors the
[upstream `docker-compose.example.yml`](https://github.com/standardnotes/server/blob/main/docker-compose.example.yml).

What it deliberately does **not** do:

- **No bundled MariaDB.** You bring your own MariaDB container —
  reuse the one you already run for Nextcloud, Vaultwarden, Photoprism,
  whatever. One DB engine for all your apps.
- **No bundled Redis.** Same logic. Reuse your existing Redis container.
- **No web client.** Standard Notes recommends the official desktop / mobile
  apps; the optional [`standardnotes/web`](https://standardnotes.com/help/self-hosting/web-app)
  image can be installed separately if you want it.

What it does do:

- **One Unraid template per concern** — `StandardNotes-Server` and
  (optionally) `StandardNotes-LocalStack`.
- **Sane defaults** taken from the upstream
  [`.env.sample`](https://github.com/standardnotes/server/blob/main/.env.sample).
- **Every secret marked `Mask="true"`** in the template, so the Unraid UI
  hides them by default.
- **Volumes match upstream paths** — `/var/lib/server/logs` and
  `/opt/server/packages/files/dist/uploads`.
- **All in German/English-friendly self-hosting docs**, focused on getting
  you to a working install without surprises.

| | **This template** | Bundled-stack templates |
|---|:---:|:---:|
| Standard Notes server image | ✅ official `standardnotes/server` | ✅ |
| MariaDB / Redis bundled | ❌ (reuse your existing MariaDB / Redis) | ✅ |
| LocalStack as a separate container | ✅ | ⚠️ embedded |
| Secrets masked in UI | ✅ | ⚠️ |
| Reverse-proxy guidance | ✅ | ⚠️ |
| Volumes match upstream paths 1:1 | ✅ | ⚠️ |

---

## 2. Architecture

```
┌──────────────────────────── Unraid Host ────────────────────────────┐
│                                                                     │
│   Reverse Proxy                                                     │
│   (SWAG / NPM / Traefik)  ──HTTPS──►  StandardNotes-Server          │
│                                       (standardnotes/server)        │
│                                       :3000  API gateway            │
│                                       :3104  files server           │
│                                            │                        │
│                              ┌─────────────┼──────────────┐         │
│                              ▼             ▼              ▼         │
│                          MariaDB       Redis        StandardNotes-  │
│                         (separate)   (separate)      LocalStack     │
│                                                       (sns, sqs)    │
└─────────────────────────────────────────────────────────────────────┘
```

The Standard Notes server image talks to:

- **MariaDB** — schema migrations run automatically on first start.
- **Redis** — cache, queues, rate limits.
- **LocalStack** — provides SNS / SQS endpoints expected by the upstream
  image, even on a single-node self-hosted setup.

---

## 3. Quick Start on Unraid

### Step 0 — Pre-flight

You will need:

- An Unraid server with **Community Applications** installed.
- A **MariaDB** container reachable from the Unraid host — see
  [§ 5](#5-database--cache).
- A **Redis** container reachable from the Unraid host — see [§ 5](#5-database--cache).
- Three 32-byte hex secrets — see [§ 4](#4-generating-secrets).
- A reverse proxy with HTTPS — see [§ 8](#8-reverse-proxy).

### Step 1 — Install the templates

The repository ships two templates:

- `templates/standardnotes-server.xml` — the Standard Notes backend.
- `templates/standardnotes-localstack.xml` — optional SNS/SQS provider.

Pull the templates into Unraid's user-template folder via the Unraid
console / SSH:

```bash
mkdir -p /boot/config/plugins/dockerMan/templates-user

curl -fsSL -o /boot/config/plugins/dockerMan/templates-user/my-StandardNotes-Server.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes/main/templates/standardnotes-server.xml

curl -fsSL -o /boot/config/plugins/dockerMan/templates-user/my-StandardNotes-LocalStack.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes/main/templates/standardnotes-localstack.xml
```

> 📌 The Unraid templates-user destination is
> `/boot/config/plugins/dockerMan/templates-user/`. Templates dropped
> there appear under **Docker → Add Container → Template → User
> templates** without restarting Docker.

### Step 2 — Start LocalStack (optional but recommended)

In the Unraid Web UI: **Docker** → **Add Container** → in the
**Template** dropdown, pick **StandardNotes-LocalStack** under
*User templates*. Hit **Apply**. No further configuration needed.

### Step 3 — Start the Standard Notes server

**Docker** → **Add Container** → pick **StandardNotes-Server** under
*User templates*. Fill in:

- **DB Host / Port / Username / Password / Name** → values of your MariaDB
  container.
- **Redis Host / Port** → values of your Redis container.
- **JWT Secret / Auth Server Encryption Key / Valet Token Secret** → three
  outputs of `openssl rand -hex 32` (see [§ 4](#4-generating-secrets)).

Hit **Apply**. First start runs the schema migrations, which can take
20–60 seconds. Watch the container log; you should eventually see the
API gateway listening on port 3000.

#### Example values from a test deployment

The values below are from one user's working `standardnotes.bottich.lol`
test setup. Use them as a shape reference — substitute your own hosts,
domain, and DB password (never paste a real password into a public
template).

| Field | Example value |
|---|---|
| Public sync URL | `https://standardnotes.bottich.lol` |
| Server container static IP | `192.168.20.38` |
| `DB Host (DB_HOST)` | `192.168.20.71` *(MariaDB on a separate VLAN)* |
| `DB Port (DB_PORT)` | `3306` |
| `DB Username (DB_USERNAME)` | `junkerderprovinz` |
| `DB Password (DB_PASSWORD)` | *(your own — generate / store in password manager)* |
| `DB Name (DB_DATABASE)` | `standardnotes` *(already created)* |
| `DB Type (DB_TYPE)` | `mysql` *(internal driver value required for MariaDB — see § 6)* |
| `Redis Host (REDIS_HOST)` | `192.168.20.72` *(no auth in this test setup)* |
| `Redis Port (REDIS_PORT)` | `6379` |
| `COOKIE_DOMAIN` | `standardnotes.bottich.lol` |
| `PUBLIC_FILES_SERVER_URL` | leave empty unless you proxy files separately — see [§ 8](#8-reverse-proxy) |

> 💡 The example above uses **LAN IPs** because the MariaDB and Redis
> containers live on a different host / VLAN than the Standard Notes
> server. When MariaDB and Redis run on the **same** Unraid host as the
> Standard Notes container on a shared bridge network, prefer the
> Unraid container names (e.g. `mariadb`, `redis`) over LAN IPs — see
> [§ 0](#0-sync-loop--duplicate-notes-guardrails) and
> [`docs/sync-loop-troubleshooting.md`](docs/sync-loop-troubleshooting.md).

### Step 4 — Reverse-proxy & connect a client

Point your reverse proxy at `http://<unraid-ip>:3000`. Open the official
[Standard Notes app](https://standardnotes.com/download) → **Advanced
options** on the sign-up screen → enter `https://your-domain/` as the
**Sync Server**. Create your account.

---

## 4. Generating Secrets

The server image requires three high-entropy secrets. Generate each with
**openssl** on any Linux/Mac host (or under WSL, or inside the Unraid
console):

```bash
openssl rand -hex 32   # AUTH_JWT_SECRET
openssl rand -hex 32   # AUTH_SERVER_ENCRYPTION_SERVER_KEY
openssl rand -hex 32   # VALET_TOKEN_SECRET
```

Each command prints a 64-character hex string. Paste them into the
matching fields in the Unraid template.

> ⚠️ **Treat these as write-once.** Rotating `AUTH_SERVER_ENCRYPTION_SERVER_KEY`
> renders existing server-side data unreadable. Back them up to your
> password manager **before** you deploy.

---

## 5. Database & Cache

This template intentionally does **not** bundle a database or Redis. Run
them as separate Unraid containers — that way one DB engine and one
Redis serve all your self-hosted apps.

### MariaDB

Use any current MariaDB container. Recommended:

- **MariaDB-Official** (Community Applications).

Create the database and user before starting the Standard Notes container:

```sql
CREATE DATABASE IF NOT EXISTS standard_notes_db
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'std_notes_user'@'%' IDENTIFIED BY 'use-a-strong-password-here';
GRANT ALL PRIVILEGES ON standard_notes_db.* TO 'std_notes_user'@'%';
FLUSH PRIVILEGES;
```

Then put the resulting `DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`,
`DB_DATABASE` values into the Unraid template fields.

> ℹ️ **About `DB_TYPE=mysql`.** Standard Notes' upstream image uses the
> TypeORM `mysql` driver string, which is the same driver used to talk
> to MariaDB. Leave `DB_TYPE=mysql` — it is an **internal driver value
> required for MariaDB**, not an instruction to install MySQL.

### Redis

Any 6.x or 7.x Redis container works. Recommended:

- **Redis-Official** (Community Applications), default port 6379, no auth.

Set `REDIS_HOST` to the Unraid container name (typically `redis`) and
`REDIS_PORT` to `6379`.

> 💡 Containers on Unraid's default `bridge` network resolve each other
> by container name. So `DB_HOST=mariadb` and `REDIS_HOST=redis` *just
> work* if those are your container names.

---

## 6. Configuration Reference

### Server template — environment variables

| Variable | Default | Description |
|---|---|---|
| `DB_HOST` | *(required)* | Hostname or IP of your MariaDB container |
| `DB_PORT` | `3306` | DB port |
| `DB_USERNAME` | `std_notes_user` | DB user |
| `DB_PASSWORD` | *(required)* | DB password |
| `DB_DATABASE` | `standard_notes_db` | DB name |
| `DB_TYPE` | `mysql` | **Internal driver value required for MariaDB.** Standard Notes / TypeORM uses the `mysql` driver string to talk to MariaDB — leave at `mysql`. |
| `REDIS_HOST` | *(required)* | Hostname of Redis container |
| `REDIS_PORT` | `6379` | Redis port |
| `CACHE_TYPE` | `redis` | Leave at `redis` |
| `AUTH_JWT_SECRET` | *(required)* | 32-byte hex — see [§ 4](#4-generating-secrets) |
| `AUTH_SERVER_ENCRYPTION_SERVER_KEY` | *(required)* | 32-byte hex — see [§ 4](#4-generating-secrets) |
| `VALET_TOKEN_SECRET` | *(required)* | 32-byte hex — see [§ 4](#4-generating-secrets) |
| `PUBLIC_FILES_SERVER_URL` | *(empty)* | Public HTTPS URL of the files server, if reverse-proxied separately |
| `COOKIE_DOMAIN` | *(empty)* | Optional but **strongly recommended** behind a reverse proxy. Must match the public sync host (e.g. `notes.example.com`). HTTPS is required for `Secure` cookies outside `localhost`. Wrong value → `No cookies provided for cookie-based session token` and possible sync loop. See [§ 0](#0-sync-loop--duplicate-notes-guardrails). |
| `STANDARDNOTES_IMAGE_TAG` | `latest` | Repository tag used by the Unraid template. Default `standardnotes/server:latest`. Pin a known-good tag if `latest` introduces session/sync regressions — change with care, see [`docs/sync-loop-troubleshooting.md`](docs/sync-loop-troubleshooting.md). |

### Ports & Volumes

| Port | Purpose |  | Volume | Purpose |
|---|---|---|---|---|
| `3000` | API gateway (HTTP, behind reverse proxy) |  | `/var/lib/server/logs` | Server logs |
| `3125 → 3104` | Files server |  | `/opt/server/packages/files/dist/uploads` | Encrypted file uploads |

A full upstream-aligned [`examples/.env.example`](examples/.env.example) is
included for reference / non-Unraid use.

---

## 7. Security

- **Always** put Standard Notes behind HTTPS — never publish port 3000
  directly. Standard Notes itself is end-to-end encrypted, but TLS still
  matters for auth tokens and metadata.
- **Back up your secrets** (the three `openssl rand -hex 32` outputs)
  alongside your DB backups. Without `AUTH_SERVER_ENCRYPTION_SERVER_KEY`
  the on-disk data is unreadable.
- **Restrict DB / Redis to the bridge network.** Don't publish their
  ports to the host unless you genuinely need external access.
- **Disable user registration** once your accounts exist, if you don't
  want the world signing up. See the
  [official self-hosting docs](https://standardnotes.com/help/self-hosting/getting-started)
  for the relevant environment variable.
- **Patch regularly** — `docker pull standardnotes/server:latest` and
  recreate, see [§ 10](#10-updating).

---

## 8. Reverse Proxy

The Standard Notes API gateway is plain HTTP on port 3000. Terminate
TLS in your reverse proxy.

### Nginx Proxy Manager (NPM)

A common Unraid setup is **Nginx Proxy Manager** running on the same
LAN as the Standard Notes container. Example using the test deployment
above:

| | Value |
|---|---|
| NPM host | `192.168.20.11` |
| Standard Notes server static IP | `192.168.20.38` |
| Public domain | `standardnotes.bottich.lol` |

In the NPM UI, **Hosts → Proxy Hosts → Add Proxy Host**:

- **Domain Names:** `standardnotes.bottich.lol`
- **Scheme:** `http`
- **Forward Hostname / IP:** `192.168.20.38`
- **Forward Port:** `3000`
- **Block Common Exploits:** on
- **Websockets Support:** on
- **SSL** tab — request a Let's Encrypt cert, **Force SSL** on,
  **HTTP/2 Support** on, **HSTS** on once you've confirmed the cert
  renews automatically.

Then in the Standard Notes template, set:

- `COOKIE_DOMAIN=standardnotes.bottich.lol`
- (optional) `PUBLIC_FILES_SERVER_URL=https://files.standardnotes.bottich.lol`
  if and only if you also create a separate proxy host for the files
  server (see below).

### Files server — separate subdomain recommended

The Standard Notes **files server** listens on its own port (`3125` on
the host → `3104` in the container) and is the upload/download
endpoint for attachments. Two options:

1. **Skip attachments (simplest).** Leave `PUBLIC_FILES_SERVER_URL`
   empty. Note creation, editing, and sync work fully — only
   attachment upload/download is disabled. This is the intended path
   for a "fully configured but minimal" template.
2. **Separate hostname (recommended if you want attachments).** Create
   a second NPM proxy host, e.g. `files.standardnotes.bottich.lol`
   → `192.168.20.38:3125`, with its own Let's Encrypt certificate.
   Then set `PUBLIC_FILES_SERVER_URL=https://files.standardnotes.bottich.lol`
   in the template.

> ⚠️ **Single-domain path proxying is not reliably supported.** The
> Standard Notes files server does not document a path-prefix mode
> (e.g. routing `/files/*` from one host to port 3104 while `/`
> hits port 3000). Some users get it working with custom NPM
> *Advanced* nginx config, but it can break across upstream image
> updates. If you want stable attachment support, give the files
> server its own host or subdomain.

### Generic reverse-proxy snippet (SWAG / nginx)

If you are not using NPM, a minimal SWAG (`nginx`) location block:

```nginx
server {
    listen 443 ssl http2;
    server_name standardnotes.bottich.lol;

    include /config/nginx/ssl.conf;

    location / {
        include /config/nginx/proxy.conf;
        resolver 127.0.0.11 valid=30s;
        set $upstream_app standardnotes-server;
        set $upstream_port 3000;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

If you reverse-proxy the **files server** under a separate hostname
(e.g. `files.standardnotes.bottich.lol`), set
`PUBLIC_FILES_SERVER_URL=https://files.standardnotes.bottich.lol` in
the template's *Public Files Server URL* field.

---

## 9. Backup & Restore

Three things to back up, in order of importance:

1. **The three secrets** (`AUTH_JWT_SECRET`,
   `AUTH_SERVER_ENCRYPTION_SERVER_KEY`, `VALET_TOKEN_SECRET`) — store in
   your password manager.
2. **The database** — daily `mysqldump` of `standard_notes_db`, kept
   off-site. The Unraid plugins
   *MariaDB-Backup* / *appdata.backup* both work.
3. **The uploads folder** — `/mnt/user/appdata/standardnotes/uploads`
   (encrypted blobs from the files server).

Restoring is the reverse: provision a fresh DB and Redis, paste the
saved secrets into the template, restore the SQL dump and the uploads
folder, start the container.

---

## 10. Updating

```bash
docker pull standardnotes/server:latest
docker stop StandardNotes-Server && docker rm StandardNotes-Server
# re-create with the same template / docker run args
```

On Unraid: **Docker** tab → click the container → **Force Update**. Your
appdata is untouched. Migrations, if any, run automatically on the next
start — watch the log for errors.

---

## 11. Troubleshooting

<details>
<summary><b>Container restarts in a loop on first start</b></summary>

- 99% of the time this is a missing or wrong DB credential. Check the
  container log: `connection refused` → wrong host/port; `Access denied
  for user` → wrong username/password.
- Make sure the database `standard_notes_db` exists and the user has
  `ALL PRIVILEGES` on it.
- The MariaDB / Redis container must be **on the same bridge network**.
  Custom `br0`s with isolated IPs need explicit DNS / IP wiring.
</details>

<details>
<summary><b>"Invalid token" / sessions break after a restart</b></summary>

- You changed `AUTH_JWT_SECRET`. Restore the old value, or accept that
  every client has to sign in again.
</details>

<details>
<summary><b>"Cannot decrypt server-side data"</b></summary>

- You changed `AUTH_SERVER_ENCRYPTION_SERVER_KEY`. Restore the old value
  from backup. Without it, the affected on-disk data is permanently
  unreadable.
</details>

<details>
<summary><b>Files server uploads fail</b></summary>

- Confirm port `3125` is reachable through the reverse proxy.
- If the files server is on a separate hostname, set
  `PUBLIC_FILES_SERVER_URL` in the template.
- Check `VALET_TOKEN_SECRET` is set (no default — required).
</details>

<details>
<summary><b>SNS / SQS errors in the log</b></summary>

- The LocalStack companion container isn't running, or the server can't
  reach it as `localstack:4566`. Either start the LocalStack template or
  configure alternative SNS/SQS endpoints in the upstream env vars.
</details>

<details>
<summary><b>Notes are being duplicated dozens of times in seconds</b></summary>

- **Stop all clients immediately**, stop the server container, do not
  keep editing. See [§ 0](#0-sync-loop--duplicate-notes-guardrails).
- Check the server log for `No cookies provided for cookie-based
  session token`, `ECONNREFUSED`, repeated `/v1/items` 401/500 calls,
  or `duplicate_of` cascades.
- Verify `REDIS_HOST` / `REDIS_PORT` and that the Redis container is
  reachable by **container name** on the same custom bridge network
  (not by LAN IP).
- Verify `COOKIE_DOMAIN` matches your public sync host and that the
  reverse proxy serves the API over **HTTPS**.
- Roll back to a known-good `standardnotes/server` tag if you recently
  upgraded — see `STANDARDNOTES_IMAGE_TAG` in the Configuration
  Reference and [`docs/sync-loop-troubleshooting.md`](docs/sync-loop-troubleshooting.md).
- **Always test with a fresh throwaway account first** before pointing
  your real, long-running client at this server.
</details>

<details>
<summary><b>Client app says "Unable to reach server"</b></summary>

- Try the API directly: `curl https://your-domain/healthcheck`. If that
  fails, the issue is in your reverse proxy, not Standard Notes.
- Mixed-content blocks: clients require HTTPS. Plain `http://` Sync
  Servers are rejected by the desktop / mobile apps.
</details>

---

## 12. Contributing / License

Pull requests welcome. Issues:
<https://github.com/junkerderprovinz/standardnotes/issues>.

> **Support link note / Hinweis zum Support-Link:** The `<Support>` tag in
> both Unraid templates currently points at the GitHub Issues tracker.
> Once an Unraid forum support thread exists, replace (or supplement) that
> URL — see [`docs/PUBLISHING.md`](docs/PUBLISHING.md) for the steps.
> Sobald ein Unraid-Forum-Thread existiert, sollte der `<Support>`-Link
> dort hin zeigen (oder zusätzlich), siehe `docs/PUBLISHING.md`.

**Licensing — dual:**

- This **wrapper repository** (Unraid templates, README, banner / icon
  artwork, example env file) is licensed under the [MIT License](LICENSE).
- **Standard Notes itself** and the bundled `standardnotes/server` Docker
  image, the LocalStack image, and any downstream MariaDB / Redis images
  retain their upstream licenses (mostly AGPL-3.0 for Standard Notes, and
  the respective licenses for everything else). When you run, redistribute
  or rebuild the resulting stack, you must comply with **all** of those
  licenses, not only with this wrapper's MIT license.

```bash
# Validate locally
xmllint --noout templates/*.xml
```

### Credits

- [**Standard Notes**](https://standardnotes.com) — the actual project &
  upstream Docker images
- [**Standard Notes server repo**](https://github.com/standardnotes/server)
  — source of the canonical `.env.sample` and `docker-compose.example.yml`
- [**LocalStack**](https://github.com/localstack/localstack) — SNS / SQS
  emulation
- [**Unraid Community Applications**](https://forums.unraid.net/topic/38582-plug-in-community-applications/)
  — the distribution channel
