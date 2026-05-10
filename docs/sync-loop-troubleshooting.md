# Sync-Loop / Duplicate Notes Troubleshooting

> Practical checklist for self-hosted Standard Notes on Unraid when
> notes start duplicating, syncing in a loop, or the client shows
> `Syncing: 0/1` indefinitely. German/English friendly.

This document is the long-form companion to
[§ 0 of the README](../README.md#0-sync-loop--duplicate-notes-guardrails).
Read that section first.

> ⚠️ If duplication is **happening right now**: stop every connected
> client, stop the `StandardNotes` container, and **do not keep
> editing notes**. Every additional edit can fan out further. Diagnose
> first, then resume.

---

## 0. Reference incidents (read once)

| Source | What it says | Status |
|---|---|---|
| [Standard Notes — *How do I clear duplicates?*](https://standardnotes.com/help/33/how-do-i-clear-duplicates) | Duplicates are **app-side conflict resolution**. The server cannot decrypt or merge notes, so the client duplicates conflicting copies to avoid silent data loss. | Current, official |
| [Forum #3635 — self-hosted session loop](https://github.com/standardnotes/forum/issues/3635) | `No cookies provided for cookie-based session token` + `/v1/items` loop traced to `COOKIE_DOMAIN` / HTTPS / reverse-proxy cookie handling and an unstable image tag. Pinning a known-good tag mitigated. | Current |
| [Legacy `standardnotes/syncing-server` #102](https://github.com/standardnotes/syncing-server/issues/102) | Stable web app + `dev`/`latest` syncing-server caused `Syncing: 0/1` and duplicate cascades. Fixed by pinning a stable tag. | **Historical / legacy.** That repo is archived; the current backend image is `standardnotes/server`. The class of failure (unstable tag vs. stable client) is still possible. |

---

## 1. Test plan — *before* you migrate any real notes

Do this on a brand-new account, not on your real one.

- [ ] Create a **fresh throwaway account** on the new self-hosted server.
- [ ] Sign in from **exactly one client** (web or desktop, not both).
- [ ] **Create one note.** Type a few words, save, wait 30 seconds.
- [ ] Refresh / reopen the client. Confirm the note exists **once**.
- [ ] Repeat with a second note, then a third. Watch the server log
      while you do — there should be no `ECONNREFUSED`, no
      `No cookies provided …`, no `duplicate_of` cascade.
- [ ] Only **after** the throwaway account has round-tripped cleanly
      for several notes: connect a second client (mobile/desktop),
      let it sync, and verify the same notes appear once each.
- [ ] **Do not** sign your real, long-history account in until the
      above is stable for at least a full sync cycle.
- [ ] **Make a decrypted export** from your existing Standard Notes
      account (cloud or old self-host) before reconnecting any real
      client. *Account → Data backups → Decrypted backup.*

If any step above misbehaves, stop and work through the rest of this
document before continuing.

---

## 2. Redis reachability

Standard Notes uses Redis for cache, queues, and rate-limiting. If
Redis is unreachable, sync de-duplication and session state get
inconsistent — a known precursor to duplicate cascades.

- [ ] In the Unraid template, **Redis Host** is set to the **IP
      address** of your Redis container (e.g. `192.168.x.x`).
      Use a static / DHCP-reserved address so the IP doesn't change
      on container restart.
- [ ] `REDIS_PORT=6379` (or whatever your Redis container actually
      listens on).
- [ ] The StandardNotes container can reach the Redis IP/port. From
      the StandardNotes container's shell:

    ```bash
    docker exec -it StandardNotes sh
    # inside the container (replace with your Redis IP):
    nc -zv 192.168.x.x 6379    # should print "open"
    # or, if nc is unavailable:
    timeout 2 sh -c 'cat </dev/tcp/192.168.x.x/6379' && echo OK
    ```

- [ ] **Expected failure mode if wrong:** the server log emits
      `ECONNREFUSED` against `redis:6379` (or your IP) on every sync
      request. The client appears to "save" notes but the server
      never confirms — clients re-send, and conflict logic on the
      app side can begin duplicating.

---

## 3. MariaDB reachability and migrations

A half-migrated database is a known cause of strange sync behaviour.

- [ ] `DB_HOST` points at the **IP address** of your MariaDB
      container (e.g. `192.168.x.x`). Use a static / DHCP-reserved
      address.
- [ ] `DB_PORT=3306`, `DB_USERNAME`, `DB_PASSWORD`, `DB_DATABASE` all
      match what you created in MariaDB.
- [ ] Database `standard_notes_db` exists with `ALL PRIVILEGES`
      granted to `std_notes_user@'%'`.
- [ ] Watch the **first start** of the server container. The log
      should show schema migrations running for ~20–60 seconds, then
      "API gateway listening on port 3000" (or similar).
- [ ] **Do not log a real client in until migrations have finished
      cleanly.** A login during migrations can end up writing partial
      session state.

---

## 4. Reverse proxy, HTTPS, and `COOKIE_DOMAIN`

This is the area most likely to cause the duplicate-notes / session-loop
class of bug ([forum #3635](https://github.com/standardnotes/forum/issues/3635)).

- [ ] The reverse proxy serves the API over **HTTPS**, not plain HTTP.
      Modern Standard Notes clients reject `http://` sync servers; in
      addition, `Secure` cookies are dropped over HTTP outside
      `localhost`, which produces the
      `No cookies provided for cookie-based session token` symptom.
- [ ] `COOKIE_DOMAIN` is set in the Unraid template and matches the
      **public** sync host. Examples:

    | Public sync URL | `COOKIE_DOMAIN` |
    |---|---|
    | `https://notes.example.com` | `notes.example.com` |
    | `https://sn.example.com` | `sn.example.com` |
    | `https://example.com/sn` (path-prefixed) | `example.com` |

- [ ] If you are doing path-prefixed proxying (`example.com/sn`),
      double-check that the proxy preserves the original
      `Host` header and sends `X-Forwarded-Proto: https`.
- [ ] If you change `COOKIE_DOMAIN` later, **all clients have to log
      in again** — old session cookies become invalid.
- [ ] **Expected failure mode if wrong:** server log fills with
      `No cookies provided for cookie-based session token` and the
      client retries `/v1/items` repeatedly (often visible as
      `Syncing: 0/1` in the client status bar).

---

## 5. Image tag — `STANDARDNOTES_IMAGE_TAG`

- [ ] By default the Unraid template uses
      `standardnotes/server:latest`. This is fine for first-time
      installs and matches upstream `docker-compose.example.yml`.
- [ ] If `latest` is **currently regressing** session or sync (forum
      #3635 was an example), pin a known-good release tag in the
      Unraid template's *Repository* field, e.g.
      `standardnotes/server:1.32.0` — replace with whatever
      `standardnotes/server` tag you have last verified.
- [ ] Or, if you prefer template-level control, set
      `STANDARDNOTES_IMAGE_TAG` (default `latest`) and reference it in
      the *Repository* field as `standardnotes/server:${STANDARDNOTES_IMAGE_TAG}`.
- [ ] **Change tags carefully.** Downgrading across schema migrations
      can leave the database in a state the older image cannot read.
      Always: full DB backup → bring server down → swap tag → bring
      server up → watch the log → only then reconnect clients.
- [ ] **Do not mix** a stable client (mobile/desktop store releases,
      stable web app) with a `dev` / `nightly` server tag. That is
      the legacy
      [syncing-server #102](https://github.com/standardnotes/syncing-server/issues/102)
      pattern and although that backend is archived, the same class
      of mismatch can still bite on `standardnotes/server`.

---

## 6. Logs to grep for

When something is wrong, these are the strings that matter. Tail the
container log:

```bash
docker logs -f StandardNotes
```

…and watch for any of the following:

| Log line / pattern | Likely cause |
|---|---|
| `No cookies provided for cookie-based session token` | `COOKIE_DOMAIN` mismatch, missing HTTPS, or proxy stripping cookies. See § 4. |
| `ECONNREFUSED` against your Redis host | Redis container is down, on a different network, or the host/port is wrong. See § 2. |
| `ECONNREFUSED` against your DB host | MariaDB is down, on a different network, or credentials/host are wrong. See § 3. |
| `/v1/items` returning 401 in a tight loop | Session not accepted by the server — almost always cookie / `COOKIE_DOMAIN` / HTTPS. See § 4. |
| `/v1/items` returning 500 in a tight loop | Server-side error — check earlier lines for the underlying exception (DB, Redis, migrations). |
| Repeated sync/update cascades, growing `updated_at` storm | Conflict-resolution loop. Stop clients immediately. |
| `duplicate_of` appearing on many notes | Confirmed duplicate cascade — the client is creating conflict copies. Stop clients, see § 7. |

---

## 7. If duplication has already started

Triage order:

1. **Stop all clients.** Sign out of mobile, desktop, and web. Don't
   just close them — sign out so they stop attempting background
   sync.
2. **Stop the server container** in the Unraid Docker tab. Leave
   MariaDB and Redis running so you can inspect state.
3. **Take a database snapshot** *before* any cleanup:

    ```bash
    docker exec -it mariadb sh -c \
      'mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" standard_notes_db' \
      > /mnt/user/appdata/standardnotes/dup-incident-$(date +%Y%m%d-%H%M).sql
    ```

4. **Inspect the logs** (see § 6) and identify the root cause —
   Redis, cookies, or image tag are by far the most common.
5. **Fix the root cause first.** Restarting the server without
   fixing the cause will reproduce the loop.
6. **Cleanup of duplicates** is documented by Standard Notes
   themselves: <https://standardnotes.com/help/33/how-do-i-clear-duplicates>.
   Use the official client procedure rather than touching the
   database directly — the server cannot decrypt notes, so
   server-side deduplication is not possible.
7. Only after the root cause is fixed and duplicates are cleaned, log
   one client back in and verify a single new note round-trips
   cleanly **before** reconnecting any other client.

---

## 8. References

- [Standard Notes — *How do I clear duplicates?*](https://standardnotes.com/help/33/how-do-i-clear-duplicates)
  — official duplicate-handling docs.
- [Forum issue #3635 — self-hosted session loop](https://github.com/standardnotes/forum/issues/3635)
  — `COOKIE_DOMAIN` / HTTPS / image-tag root cause.
- [Legacy `standardnotes/syncing-server` issue #102](https://github.com/standardnotes/syncing-server/issues/102)
  — historical stable-client / dev-server mismatch causing
  `Syncing: 0/1` and duplicates. Archived backend; class of failure
  still relevant.
- [Standard Notes self-hosting docs](https://standardnotes.com/help/self-hosting/getting-started)
- [`standardnotes/server` upstream `.env.sample`](https://github.com/standardnotes/server/blob/main/.env.sample)
