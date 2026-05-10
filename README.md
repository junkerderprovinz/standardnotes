<h1 align="center">Standard Notes Server for Unraid</h1>

<a href="https://standardnotes.com">
  <img src=".github/assets/banner.svg" alt="Standard Notes Server for Unraid" width="100%">
</a>

<p align="center">
  <a href="#3-quick-start-on-unraid"><img src="https://img.shields.io/badge/Unraid-Community%20Template-f15a2c?style=for-the-badge&logo=unraid&logoColor=white" alt="Unraid Community Template"></a>
  <a href="https://standardnotes.com"><img src="https://img.shields.io/badge/Standard%20Notes-self--hosted-086DD7?style=for-the-badge&logo=standardnotes&logoColor=white" alt="Standard Notes"></a>
  <a href="https://hub.docker.com/r/standardnotes/server"><img src="https://img.shields.io/badge/Docker-standardnotes%2Fserver-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker image"></a>
  <a href="#5-database--cache"><img src="https://img.shields.io/badge/Database-MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white" alt="MariaDB"></a>
  <a href="#5-database--cache"><img src="https://img.shields.io/badge/Cache-Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white" alt="Redis"></a>
  <a href="https://github.com/junkerderprovinz/standardnotes-server/actions/workflows/lint.yml"><img src="https://img.shields.io/github/actions/workflow/status/junkerderprovinz/standardnotes-server/lint.yml?branch=main&style=for-the-badge&logo=githubactions&logoColor=white&label=lint" alt="Lint"></a>
  <a href="https://github.com/junkerderprovinz/standardnotes-server/actions/workflows/build.yml"><img src="https://img.shields.io/github/actions/workflow/status/junkerderprovinz/standardnotes-server/build.yml?branch=main&style=for-the-badge&logo=githubactions&logoColor=white&label=build" alt="Build"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge" alt="License: MIT"></a>
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
- **Wrong host or unreachable network** for `DB_HOST` / `REDIS_HOST`.
  This template asks for IP addresses (e.g. `192.168.x.x`) — they
  are unambiguous across Unraid's bridge / `br0` / VLAN setups. Make
  sure the IP is static (DHCP reservation or fixed) so it does not
  change on container restart.
- **Image / client version mismatch.** The legacy
  [`standardnotes/syncing-server` issue
  #102](https://github.com/standardnotes/syncing-server/issues/102)
  documents how mixing a stable web app with a development
  syncing-server caused `Syncing: 0/1` and duplicate cascades. That
  repo is **archived and historical** — the current image is
  `standardnotes/server` — but the same class of mismatch can still
  happen if you pin to a broken `latest`.

### `https://` — where it belongs and where it does NOT

A misplaced `https://` in `COOKIE_DOMAIN` is one of the most common
self-host pitfalls and has been observed to cause the duplicate-loop
class of bug. Use this matrix:

| Field | Value type | Includes `https://`? | Example |
|---|---|---|---|
| `COOKIE_DOMAIN` (env / template) | **Bare domain only** | ❌ **No** | `standardnotesserver.mydomain.tld` |
| `PUBLIC_FILES_SERVER_URL` (env / template) | **Full HTTPS URL** | ✅ **Yes** | `https://files.standardnotesserver.mydomain.tld` |
| Custom Sync Server (in client / web app) | **Full HTTPS URL** | ✅ **Yes** | `https://standardnotesserver.mydomain.tld` |

If you accidentally set `COOKIE_DOMAIN=https://standardnotesserver.mydomain.tld`,
the server emits cookies for the literal string `https://...` which no
browser will accept — sessions silently break, the client retries
`/v1/items`, and conflict-resolution on the client can cascade into
duplicates.

### Emergency checklist — duplicates happening RIGHT NOW

> Stop first. Diagnose second. Edit nothing.

1. **Stop every connected client immediately.** Sign out (don't just
   close) on web UI, desktop, mobile. Background sync keeps fanning the
   loop out otherwise.
2. **Stop the StandardNotesServer container.** Unraid → Docker → stop.
   Leave MariaDB and Redis running so you can inspect state.
3. **Do NOT reconnect any existing client.** Especially not your
   long-history account. Each new edit during a loop can spawn more
   duplicates.
4. **Check Redis.** From inside the container shell:
   `nc -zv <redis-ip> 6379`. Tail the server log for `ECONNREFUSED`
   against your Redis host.
5. **Check `COOKIE_DOMAIN`.** Verify it is a **bare domain** —
   `standardnotesserver.mydomain.tld`, not `https://...`, not a URL,
   no trailing slash. Verify the Custom Sync Server URL in the client
   is the **full HTTPS URL** `https://standardnotesserver.mydomain.tld`.
6. **If the test/throwaway account is the one that duplicated:**
   delete it (or drop and recreate the database / wipe the test
   account) before bringing the server back up. Carrying a
   already-corrupted account into a "fixed" deployment will reproduce
   the cascade as soon as a client reconnects.

Full triage: [`docs/sync-loop-troubleshooting.md`](docs/sync-loop-troubleshooting.md).

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
- **No bundled web client.** Standard Notes recommends the official
  desktop / mobile apps. If you want a browser client too, the
  optional [`standardnotes/web`](https://standardnotes.com/help/self-hosting/web-app)
  image is shipped as a **separate** Unraid template in the companion
  repo
  [`junkerderprovinz/standardnotes-webui`](https://github.com/junkerderprovinz/standardnotes-webui)
  (container name `StandardNotes`). Install it on its own and point it
  at this server via your reverse proxy.

What it does do:

- **One Unraid template per concern** — `StandardNotesServer` and
  (optionally) `StandardNotes-LocalStack`. The browser client lives in
  the companion repo as its own `StandardNotes` template.
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

### Scope: backend only, no paid-feature unlocks, no AiO image

This template ships **only the official Standard Notes backend**
(`standardnotes/server`) plus the optional LocalStack companion. A few
points users frequently ask about:

- **Paid Standard Notes features are not unlocked by self-hosting.**
  Subscription-only client features (extended editors, advanced themes,
  Files quota, Listed, etc.) are gated by Standard Notes' own licensing
  / subscription checks, which live in the official clients and the
  upstream server. This template **does not** patch, bypass, or
  otherwise modify those checks — it deploys the upstream image as-is.
  Self-hosting gives you data ownership and a free Sync server; it does
  not turn a free account into a paid one.
- **Web UI is intentionally a separate, optional container.** If you
  want a browser client in addition to the desktop / mobile apps, run
  the official [`standardnotes/web`](https://standardnotes.com/help/self-hosting/web-app)
  image as its **own** Unraid container and point it at this server via
  your reverse proxy. The companion template
  [`junkerderprovinz/standardnotes-webui`](https://github.com/junkerderprovinz/standardnotes-webui)
  ships exactly that — container name `StandardNotes`, default host
  port `3001` to avoid the backend's `:3000`. Keeping web and server
  separate mirrors upstream's own `docker-compose.example.yml`, makes
  upgrades independent, and avoids the maintenance burden of a custom
  rebuilt all-in-one image.
- **No All-in-One (AiO) container.** An AiO image bundling server +
  web + DB + cache is **not** planned for the first Community
  Applications release. AiO would require a custom rebuilt image
  (diverging from upstream tags), would re-bundle MariaDB / Redis that
  most Unraid users already run, and would have to be re-released on
  every upstream component bump. This wrapper deliberately stays close
  to upstream so updates are just a tag change.

---

## 2. Architecture

```
┌──────────────────────────── Unraid Host ────────────────────────────┐
│                                                                     │
│   Reverse Proxy                                                     │
│   (SWAG / NPM / Traefik)  ──HTTPS──►  StandardNotesServer           │
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
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes-server/main/templates/standardnotes-server.xml

curl -fsSL -o /boot/config/plugins/dockerMan/templates-user/my-StandardNotes-LocalStack.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes-server/main/templates/standardnotes-localstack.xml
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

**Docker** → **Add Container** → pick **StandardNotesServer** under
*User templates*. Fill in:

- **DB Host / Port / Username / Password / Name** → values of your MariaDB
  container.
- **Redis Host / Port** → values of your Redis container.
- **JWT Secret / Auth Server Encryption Key / Valet Token Secret** → three
  outputs of `openssl rand -hex 32` (see [§ 4](#4-generating-secrets)).

Hit **Apply**. First start runs the schema migrations, which can take
20–60 seconds. Watch the container log; you should eventually see the
API gateway listening on port 3000.

#### Example values (shape reference)

Substitute your own hosts, domain, and DB password (never paste a real
password into a public template).

| Field | Example value |
|---|---|
| Public sync URL | `https://standardnotesserver.mydomain.tld` |
| Server container static IP | `192.168.x.x` |
| MariaDB Host | `192.168.x.x` |
| MariaDB Port | `3306` |
| MariaDB User | `std_notes_user` |
| MariaDB Password | *(your own — store in password manager)* |
| MariaDB Database Name | `standard_notes_db` *(already created)* |
| Database Driver (`DB_TYPE`) | `mysql` *(internal driver value required for MariaDB — see § 6)* |
| Redis Host | `192.168.x.x` |
| Redis Port | `6379` |
| Cookie Domain (`COOKIE_DOMAIN`) | `standardnotesserver.mydomain.tld` *(bare domain — no `https://`)* |
| Public Files Server URL (`PUBLIC_FILES_SERVER_URL`) | *(optional)* `https://files.standardnotesserver.mydomain.tld` *(full HTTPS URL — includes `https://`)* — leave empty to skip attachments — see [§ 8](#8-reverse-proxy) |
| Custom Sync Server *(entered in the client / web app)* | `https://standardnotesserver.mydomain.tld` *(full HTTPS URL)* |

> 💡 This template asks for **IP addresses** for the MariaDB and Redis
> hosts. IPs are unambiguous across Unraid's bridge / `br0` / VLAN
> setups and do not silently break when Docker bridge names get
> reassigned. Use a static / DHCP-reserved address so the IP doesn't
> change on container restart.

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

Set the Redis Host field to the **IP address** of your Redis container
(e.g. `192.168.x.x`) and `REDIS_PORT` to `6379`.

> 🔒 **Redis security.** This community template intentionally does
> **not** expose a Redis password field. The official
> `standardnotes/server` image does not document a Redis-auth env var,
> and adding one without verifying upstream support tends to break
> installs. Instead, keep Redis on a **trusted VLAN or private bridge**
> and firewall it off from the public internet. If your particular
> image fork supports Redis auth, verify the env-var name against its
> docs first, then add it as a custom Variable in the template.

---

## 6. Configuration Reference

### Server template — environment variables

| Variable | Default | Description |
|---|---|---|
| `DB_HOST` | *(required)* | IP address of your MariaDB container, e.g. `192.168.x.x` |
| `DB_PORT` | `3306` | DB port |
| `DB_USERNAME` | `std_notes_user` | DB user |
| `DB_PASSWORD` | *(required)* | DB password |
| `DB_DATABASE` | `standard_notes_db` | DB name |
| `DB_TYPE` | `mysql` | **Internal driver value required for MariaDB.** Standard Notes / TypeORM uses the `mysql` driver string to talk to MariaDB — leave at `mysql`. |
| `REDIS_HOST` | *(required)* | IP address of your Redis container, e.g. `192.168.x.x` |
| `REDIS_PORT` | `6379` | Redis port |
| `CACHE_TYPE` | `redis` | Leave at `redis` |
| `AUTH_JWT_SECRET` | *(required)* | 32-byte hex — see [§ 4](#4-generating-secrets) |
| `AUTH_SERVER_ENCRYPTION_SERVER_KEY` | *(required)* | 32-byte hex — see [§ 4](#4-generating-secrets) |
| `VALET_TOKEN_SECRET` | *(required)* | 32-byte hex — see [§ 4](#4-generating-secrets) |
| `PUBLIC_FILES_SERVER_URL` | *(empty)* | **Full HTTPS URL** of the files server (includes `https://`), e.g. `https://files.standardnotesserver.mydomain.tld`. Set only when you reverse-proxy the files server on its own subdomain. |
| `COOKIE_DOMAIN` | *(empty)* | **Bare domain only — no protocol, no `https://`, no path.** Example: `standardnotesserver.mydomain.tld` ✅. Wrong: `https://standardnotesserver.mydomain.tld` ❌ (that's a URL — it breaks session cookies). The Custom Sync Server URL entered in clients is a separate value and *is* a full HTTPS URL: `https://standardnotesserver.mydomain.tld`. HTTPS is required for `Secure` cookies outside `localhost`. Wrong value → `No cookies provided for cookie-based session token` and a possible sync loop. See [§ 0](#0-sync-loop--duplicate-notes-guardrails). |
| `STANDARDNOTES_IMAGE_TAG` | `latest` | Repository tag used by the Unraid template. Default `standardnotes/server:latest`. Pin a known-good tag if `latest` introduces session/sync regressions — change with care, see [`docs/sync-loop-troubleshooting.md`](docs/sync-loop-troubleshooting.md). |

### Ports & Volumes

| Host Port | Container Port | Purpose |
|---|---|---|
| `3000` | `3000` | API gateway (HTTP, behind reverse proxy) |
| `3125` | `3104` | Files server. **Forward your reverse proxy to host port `3125`**, not `3104`. The internal upstream files service listens on `3104` *inside* the container; `3125` is the published host port (per upstream `docker-compose.example.yml`). The Unraid template's *Files Server Port* field reflects this: container target `3104`, default host port `3125`. |

| Volume | Purpose |
|---|---|
| `/var/lib/server/logs` | Server logs |
| `/opt/server/packages/files/dist/uploads` | Encrypted file uploads |

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
LAN as the Standard Notes container. Example shape (substitute your
own LAN IPs and domain):

| | Value |
|---|---|
| NPM host | `192.168.x.x` |
| Standard Notes server static IP | `192.168.x.x` |
| Public domain | `standardnotesserver.mydomain.tld` |

In the NPM UI, **Hosts → Proxy Hosts → Add Proxy Host**:

- **Domain Names:** `standardnotesserver.mydomain.tld`
- **Scheme:** `http`
- **Forward Hostname / IP:** `192.168.x.x` *(StandardNotesServer container IP)*
- **Forward Port:** `3000`
- **Block Common Exploits:** on
- **Websockets Support:** on
- **SSL** tab — request a Let's Encrypt cert, **Force SSL** on,
  **HTTP/2 Support** on, **HSTS** on once you've confirmed the cert
  renews automatically.

Then in the Standard Notes template, set:

- `COOKIE_DOMAIN=standardnotesserver.mydomain.tld`
  — **bare domain only**, no `https://`, no trailing slash.
- (optional) `PUBLIC_FILES_SERVER_URL=https://files.standardnotesserver.mydomain.tld`
  — **full HTTPS URL** (with `https://`); set only if you also create a
  separate proxy host for the files server (see below).

In the Standard Notes desktop / mobile / web client, the **Custom Sync
Server** field is also a full HTTPS URL: `https://standardnotesserver.mydomain.tld`.

### Files server — is a second subdomain required?

**Short answer:** No, a second subdomain is not strictly required, but
for any **fully configured public** setup with attachments it is the
recommended path.

The Standard Notes **files server** listens on container port `3104`
internally; the upstream `docker-compose.example.yml` publishes that as
host port `3125` (mapping `3125:3104`). Forward your reverse proxy to
**host port 3125** — never to `3104` directly, since `3104` is the
internal container port and is not exposed by this template. The
official Docker docs expose the sync server on `:3000` and the files
server on `:3125` and configure `PUBLIC_FILES_SERVER_URL` to a separate
URL — that is the upstream shape. Two options:

1. **Skip attachments (simplest).** Leave **Public Files Server URL**
   empty. Note creation, editing, and sync all work — only attachment
   upload/download is disabled. This is the intended path for a
   minimal community-template install.
2. **Two NPM proxy hosts / two subdomains (recommended for full
   attachment support).** Add a second NPM proxy host, e.g.
   `files.standardnotesserver.mydomain.tld` → `192.168.x.x:3125` (the
   StandardNotesServer container IP), with its own Let's Encrypt
   certificate and **Force SSL** on. Then set
   **Public Files Server URL** = `https://files.standardnotesserver.mydomain.tld`
   in the template.

> ⚠️ **Same-domain / path-prefix proxying is not recommended for this
> community template.** The upstream image documents the two-port,
> two-URL split (sync on `:3000`, files on `:3125` with
> `PUBLIC_FILES_SERVER_URL`). Some operators get a single-domain
> `/files/*` setup working with custom NPM *Advanced* nginx config,
> but it can break across upstream image updates and is not part of
> the supported template default. For stable attachment support,
> give the files server its own subdomain.

### Generic reverse-proxy snippet (SWAG / nginx)

If you are not using NPM, a minimal SWAG (`nginx`) location block:

```nginx
server {
    listen 443 ssl http2;
    server_name standardnotesserver.mydomain.tld;

    include /config/nginx/ssl.conf;

    location / {
        include /config/nginx/proxy.conf;
        resolver 127.0.0.11 valid=30s;
        set $upstream_app 192.168.x.x;
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
(e.g. `files.standardnotesserver.mydomain.tld`), set
`PUBLIC_FILES_SERVER_URL=https://files.standardnotesserver.mydomain.tld` in
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
docker stop StandardNotesServer && docker rm StandardNotesServer
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
- Verify `REDIS_HOST` / `REDIS_PORT` point at the right **IP** and
  that the Redis container is actually reachable from the
  StandardNotesServer container (`nc -zv <ip> 6379`).
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
<https://github.com/junkerderprovinz/standardnotes-server/issues>.

> **Support link note / Hinweis zum Support-Link:** The `<Support>` tag in
> both Unraid templates currently points at the GitHub Issues tracker.
> Once an Unraid forum support thread exists, replace (or supplement) that
> URL — see [`docs/PUBLISHING.md`](docs/PUBLISHING.md) for the steps.
> Sobald ein Unraid-Forum-Thread existiert, sollte der `<Support>`-Link
> dort hin zeigen (oder zusätzlich), siehe `docs/PUBLISHING.md`.

**License:** [MIT](LICENSE).

This wrapper repository (Unraid templates, README, banner / icon artwork,
example env file) is MIT-licensed. The upstream `standardnotes/server`,
LocalStack, MariaDB and Redis images retain their own upstream licenses
(Standard Notes: AGPL-3.0; LocalStack: Apache-2.0; MariaDB: GPL-2.0;
Redis: see upstream) — comply with those when running or redistributing
the resulting stack.

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
