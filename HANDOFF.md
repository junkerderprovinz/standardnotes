# HANDOFF — standardnotes (Unraid templates)

Local working directory. **Publishing remains paused** — nothing has
been pushed, no remote has been created, and no Community Applications
submission has been made. Do not push, create a repo, or publish until
the user explicitly approves.

> Repo name change: the published GitHub repository is
> `junkerderprovinz/standardnotes` (no `-unraid` suffix). All
> in-template `<TemplateURL>`, `<Icon>`, `<Support>`, README install
> commands, and `docs/PUBLISHING.md` URLs have been updated
> accordingly. Local working directory remains
> `standardnotes-unraid/` for now and the LICENSE credit line still
> reads `standardnotes-unraid contributors` — neither is user-facing
> on the published repo.

## Latest pass — MariaDB-only wording, NPM example, repo URL switch

This pass folds in the user's `standardnotes.bottich.lol` test setup
and tightens public wording to MariaDB-only.

- **MariaDB-only wording, public-facing.** Removed every
  `MariaDB / MySQL`, `MariaDB or MySQL 8`, `mysql:8` mention from the
  README, server XML template, configuration docs, sync-loop
  troubleshooting doc, and `examples/.env.example`. The only
  remaining references to `mysql` are:
  - The `DB_TYPE=mysql` value itself, with an inline note explaining
    it is the **internal driver value required for MariaDB** (TypeORM
    uses the `mysql` driver string to talk to MariaDB).
  - `mysqldump` / `MYSQL_ROOT_PASSWORD` in the duplicate-incident
    triage snippet — those are the actual MariaDB CLI tool / env var
    names.
- **Example values for the user's test deployment.** Added a "Example
  values from a test deployment" subsection in README § 3 with the
  shape of the user's setup: domain `standardnotes.bottich.lol`,
  static container IP `192.168.20.38`, MariaDB `192.168.20.71`
  (database `standardnotes`, user `junkerderprovinz`), Redis
  `192.168.20.72` (no auth in this test setup), `COOKIE_DOMAIN`
  matching the public host. **No DB password baked in** — the table
  explicitly says "your own — generate / store in password manager".
- **NPM reverse-proxy guidance.** README § 8 rewritten to lead with
  Nginx Proxy Manager at `192.168.20.11` proxying
  `standardnotes.bottich.lol → 192.168.20.38:3000`, with a separate
  subsection on the **files server**: skip attachments for the
  simplest path, or create a separate subdomain (e.g.
  `files.standardnotes.bottich.lol`) for full attachment support.
  Single-domain path proxying (one host, `/files/*` prefix) is
  explicitly called out as **not reliably supported** by the upstream
  files server.
- **Server XML — DB / Redis host descriptions relaxed for off-host
  deployments.** `DB_HOST` / `REDIS_HOST` no longer demand container
  names absolutely; they now say: prefer container names when DB /
  Redis share the Unraid host on a custom Docker bridge, but a
  static LAN IP (e.g. `192.168.20.71` / `192.168.20.72`) is fine
  when DB / Redis live on a different host / VLAN. Sync-loop
  troubleshooting doc updated to match.
- **`DB_TYPE` description hardened.** Marked as
  *"INTERNAL DRIVER VALUE REQUIRED FOR MARIADB"* in the server XML
  config, the README configuration table, and `docs/configuration.md`,
  with the explicit "do not change; the templates and docs assume
  MariaDB" note.
- **Repo URL switch — `standardnotes-unraid` → `standardnotes`.**
  Updated `<TemplateURL>`, `<Icon>`, `<Support>` in both XML
  templates; the README install commands and contribute / issues
  link; and `docs/PUBLISHING.md` (clone command, raw-URL smoke test,
  install snippet). Added an explicit "Install command
  (templates-user destination)" block in `docs/PUBLISHING.md`
  pointing at the canonical raw URLs and the
  `/boot/config/plugins/dockerMan/templates-user/` destination.
- **`examples/.env.example`.** `DB_HOST` comment updated to the
  container-name-preferred / LAN-IP-allowed wording. `COOKIE_DOMAIN`
  example switched to `standardnotes.bottich.lol`.
  `PUBLIC_FILES_SERVER_URL` example switched to
  `https://files.standardnotes.bottich.lol` with the
  attachments-optional / single-domain caveat.

### Files changed in this pass

```
standardnotes/
├── README.md                              # MariaDB-only wording,
│                                          # NPM § 8 rewrite, example
│                                          # values table, repo URLs,
│                                          # DB_TYPE note
├── HANDOFF.md                             # this update
├── docs/
│   ├── PUBLISHING.md                      # repo URLs, install
│   │                                      # snippet
│   ├── configuration.md                   # MariaDB-only DB_TYPE
│   │                                      # note
│   └── sync-loop-troubleshooting.md       # DB host wording for
│                                          # cross-host setups
├── templates/
│   ├── standardnotes-server.xml           # MariaDB-only Overview,
│   │                                      # DB_TYPE wording, DB /
│   │                                      # Redis host wording, repo
│   │                                      # URLs, COOKIE_DOMAIN
│   │                                      # example, files-server
│   │                                      # caveat
│   └── standardnotes-localstack.xml       # repo URLs
└── examples/
    └── .env.example                       # MariaDB wording, example
                                           # COOKIE_DOMAIN /
                                           # PUBLIC_FILES_SERVER_URL
```

### Validation (this pass)

```
$ python3 -c "import xml.etree.ElementTree as ET; \
    [print('ok', f) or ET.parse(f) for f in [ \
      'templates/standardnotes-server.xml', \
      'templates/standardnotes-localstack.xml', \
      '.github/assets/banner.svg', \
      '.github/assets/icon.svg']]"
ok templates/standardnotes-server.xml
ok templates/standardnotes-localstack.xml
ok .github/assets/banner.svg
ok .github/assets/icon.svg

$ grep -rniE "MariaDB\s*/\s*MySQL|MariaDB or MySQL" .
(no matches — all MariaDB-or-MySQL alternatives removed from
 user-facing wording)

$ grep -rni "MySQL" templates README.md docs examples
(only matches are the explanatory `DB_TYPE=mysql` driver note and
 `mysqldump` / `MYSQL_ROOT_PASSWORD` in the triage snippet — both
 expected and intentional)

$ grep -rnE "REPLACE_WITH_[A-Z_]+" templates README.md docs
(no matches — clean)

$ grep -rn "standardnotes-unraid" templates README.md docs examples
(no matches — repo URL switch complete in user-facing files;
 LICENSE credit line and the local working directory name still
 read "standardnotes-unraid" but neither is user-facing on the
 published repo)
```

---

## Previous pass — Sync-Loop / Duplicate-Notes safety

This pass focuses on **user-reported duplicate notes** in a previous
self-hosted install ("Jedesmal wenn ich eine Notiz in der App
erstellt hatte, hat es diese Notiz in sekunden fast hundertfach
repliziert."). The repo is now hardened with:

- **README § 0** — new prominent "Sync-Loop / Duplicate Notes
  Guardrails" section at the top of the README, before any quick-start
  steps. Lists common causes (sync conflicts, Redis ECONNREFUSED,
  `COOKIE_DOMAIN`, reverse-proxy HTTPS/cookies, image/client mismatch,
  IPs vs container names) and warns to test with a fresh account
  before migrating real notes.
- **`docs/sync-loop-troubleshooting.md`** — full Unraid-tailored
  checklist covering throwaway-account testing, Redis reachability
  (incl. `nc -zv` test from inside the container), MariaDB
  migrations, `COOKIE_DOMAIN` / HTTPS table, `STANDARDNOTES_IMAGE_TAG`
  pinning, log strings to grep for, and a triage order if duplication
  has already started.
- **README §11 troubleshooting** — added a `<details>` block "Notes
  are being duplicated dozens of times in seconds" that links into the
  guardrails section and the new docs file.
- **`templates/standardnotes-server.xml`**:
  - Added optional `COOKIE_DOMAIN` env field with an explicit
    description warning about the
    `No cookies provided for cookie-based session token` symptom and
    HTTPS requirement.
  - Tightened `REDIS_HOST` / `REDIS_PORT` / `DB_HOST` descriptions to
    require container names on a shared bridge network and explicitly
    discourage LAN IPs. Both Redis fields stay `Required="true"`.
  - Added a "SYNC-LOOP / DUPLICATE-NOTES WARNING" paragraph and an
    image-tag pinning note to `<Overview>`. The tag is controlled via
    the existing `<Repository>` field (default
    `standardnotes/server:latest`), with a documented option to pin
    a known-good tag.
- **`examples/.env.example`** — documented `COOKIE_DOMAIN` with the
  same warning context.
- **README §6 configuration table** — added rows for `COOKIE_DOMAIN`
  and `STANDARDNOTES_IMAGE_TAG`.
- **De-emphasised Breeze Dark** wording: removed `breeze-dark` from
  the suggested GitHub topics, rephrased the description, and added
  notes in HANDOFF.md and `docs/PUBLISHING.md` clarifying that the
  dark colour palette is **repo-decorative** for the GitHub README
  only — Standard Notes itself has no Breeze / KDE relationship.
- **README banner** (`.github/assets/banner.svg`): refreshed to a
  clean centred lockup on a white background — official Standard
  Notes mark plus the "Standard Notes" wordmark only. No "powered by
  Proton" / "by Proton" wording in the visual asset. Validated with
  `python3 -c "import xml.etree.ElementTree as ET; ET.parse(...)"`
  and grepped the repo: zero matches for `proton` / `powered by`.

The README banner/layout still mirrors the
[`junkerderprovinz/krusader`](https://github.com/junkerderprovinz/krusader)
visual style (centred header, badge row, numbered TOC, `<details>`
troubleshooting blocks). That is **layout / presentation only**, not
product branding.

### Files changed in this pass

```
standardnotes-unraid/
├── README.md                              # § 0 added, §6 table extended, §11 entry added
├── HANDOFF.md                             # this rewrite — safety summary, publish-paused note
├── docs/
│   ├── sync-loop-troubleshooting.md       # NEW — Unraid-tailored checklist
│   └── PUBLISHING.md                      # Breeze Dark wording de-emphasised
├── templates/
│   └── standardnotes-server.xml           # COOKIE_DOMAIN added, Redis/DB descriptions hardened, Overview warning
└── examples/
    └── .env.example                       # COOKIE_DOMAIN documented
```

### Validation (this pass)

```
$ python3 -c "import xml.etree.ElementTree as ET; \
    [print('ok', f) or ET.parse(f) for f in [ \
      'templates/standardnotes-server.xml', \
      'templates/standardnotes-localstack.xml', \
      '.github/assets/banner.svg', \
      '.github/assets/icon.svg']]"
ok templates/standardnotes-server.xml
ok templates/standardnotes-localstack.xml
ok .github/assets/banner.svg
ok .github/assets/icon.svg

$ grep -rnEi "breeze[ -]?dark" templates README.md examples
(no matches — product wording removed; only the meta-notes in
 HANDOFF.md / docs/PUBLISHING.md mention it, and only to clarify
 that it is NOT relevant to Standard Notes.)

$ grep -rnE "REPLACE_WITH_[A-Z_]+" templates README.md docs
(no matches — clean)
```

### Publishing status

**Paused.** The user has not approved publishing. Do **not**:

- run `git push`, `git remote add`, or `gh repo create`,
- submit to `Squidly271/AppFeed`,
- open an Unraid forum support thread on the user's behalf,
- tag a release.

`docs/PUBLISHING.md` remains a checklist for **when the user later
decides to publish**, not a green light to act on.

---

## Original handoff context (carried forward)

## Suggested GitHub repo metadata

- **Repo name:** `standardnotes-unraid`
- **Owner:** `junkerderprovinz`
- **Description:** *Unraid Community Template for the Standard Notes self-hosted backend. MariaDB and Redis as separate containers, with sync-loop / duplicate-notes guardrails.*
- **Topics / tags:** `unraid`, `unraid-template`, `standard-notes`, `self-hosted`, `docker`, `community-applications`, `mariadb`, `redis`

> Note: Standard Notes is **not** related to or themed around Breeze Dark.
> The repo's banner/README uses a dark palette purely for GitHub
> presentation — it has no bearing on Standard Notes itself, and
> `breeze-dark` is **not** a suitable product topic for this repo.

## Current placeholder state

- `REPLACE_WITH_USER` — **resolved.** Replaced everywhere with
  `junkerderprovinz` in `templates/*.xml` and `README.md`.
- `REPLACE_WITH_FORUM_THREAD` — **deferred.** No forum thread exists
  yet, and inventing a URL would produce a broken `<Support>` link in
  Community Applications. Both templates' `<Support>` tag now points at
  the GitHub Issues tracker
  (`https://github.com/junkerderprovinz/standardnotes-unraid/issues`)
  as a working fallback. The README and `docs/PUBLISHING.md` both
  document that this should be replaced (or supplemented) with the
  Unraid forum thread URL once one exists — see
  [`docs/PUBLISHING.md`](docs/PUBLISHING.md) § 3.

## Files created / changed in this pass

```
standardnotes-unraid/
├── .github/
│   ├── assets/
│   │   ├── banner.svg                # unchanged
│   │   └── icon.svg                  # unchanged
│   └── workflows/
│       └── validate.yml              # NEW — XML/SVG well-formedness, placeholder guard
├── templates/
│   ├── standardnotes-server.xml      # placeholders replaced
│   └── standardnotes-localstack.xml  # placeholders replaced
├── examples/
│   └── .env.example                  # unchanged
├── docs/
│   ├── configuration.md              # unchanged
│   └── PUBLISHING.md                 # NEW — CA submission checklist
├── LICENSE                           # unchanged
├── README.md                         # placeholders replaced + Support-link note
└── HANDOFF.md                        # this file (rewritten)
```

## CI / validation

`.github/workflows/validate.yml` runs on every push and PR to `main`:

1. Parses `templates/*.xml` and `.github/assets/*.svg` with Python's
   `xml.etree.ElementTree` — no `xmllint` dependency, so it runs on a
   stock `ubuntu-latest` runner without any apt installs.
2. Greps for any leftover `REPLACE_WITH_*` placeholders in
   `templates/`, `README.md`, and `docs/` and fails the job if any
   are found.

Local equivalent:

```bash
python3 -c "import xml.etree.ElementTree as ET; \
  [ET.parse(f) for f in ['templates/standardnotes-server.xml', \
                         'templates/standardnotes-localstack.xml', \
                         '.github/assets/banner.svg', \
                         '.github/assets/icon.svg']]"
grep -rnE "REPLACE_WITH_[A-Z_]+" templates README.md docs
```

## Validation run (this pass)

```
$ python3 -c "import xml.etree.ElementTree as ET; \
    [print('ok', f) or ET.parse(f) for f in [ \
      'templates/standardnotes-server.xml', \
      'templates/standardnotes-localstack.xml', \
      '.github/assets/banner.svg', \
      '.github/assets/icon.svg']]"
ok templates/standardnotes-server.xml
ok templates/standardnotes-localstack.xml
ok .github/assets/banner.svg
ok .github/assets/icon.svg

$ grep -rnE "REPLACE_WITH_[A-Z_]+" templates README.md docs
(no matches — clean)
```

No real secrets are present in the working tree (`examples/.env.example`
contains placeholders only; `templates/*.xml` mark all secret fields
`Mask="true"` with empty defaults).

## Design decisions / assumptions (carried forward)

- **Two templates, no bundling.** Per the user's explicit constraint,
  the Standard Notes template references — but does not include —
  MariaDB or Redis. LocalStack is split into its own optional companion
  template so one Unraid XML still represents one Docker container,
  matching CA conventions.
- **Image is `standardnotes/server:latest`** as in the upstream
  `docker-compose.example.yml`. No custom build, no GHCR mirror.
- **Volumes match upstream paths 1:1**
  (`/var/lib/server/logs`, `/opt/server/packages/files/dist/uploads`)
  so upstream documentation applies unchanged.
- **Ports**: `3000:3000` for the API gateway, `3125:3104` for the files
  server (the asymmetric host:container mapping is verbatim from
  upstream).
- **All three secrets** (`AUTH_JWT_SECRET`,
  `AUTH_SERVER_ENCRYPTION_SERVER_KEY`, `VALET_TOKEN_SECRET`) are
  exposed as required, masked fields with explicit `openssl rand -hex
  32` guidance.
- **DB / Redis hosts are `Required="true"` with empty defaults** — the
  user is forced to fill them in, since we don't know the names of
  their existing MariaDB / Redis containers.
- **No promise of upstream support** — README, the server template's
  `<Overview>`, and `LICENSE` all explicitly call out that this is an
  unofficial wrapper and link to the official docs.
- **No web client.** The optional `standardnotes/web` image is
  mentioned in the README but deliberately not templated.
- **Banner is SVG**, not PNG, to keep the repo small and to match the
  reference repo style.

## Style conventions adopted from the reference repo

- Centred `<h1>`, banner image inside an anchor, centred shields.io
  badge row, centred italic subtitle — same layout as
  `junkerderprovinz/krusader`.
- 12-section TOC with the same nesting / numbering style.
- Comparison table near the top.
- "Quick Start" broken into numbered steps.
- `<details>` blocks for the troubleshooting section.
- Dual-license notice at the bottom, plus a Credits subsection.
- `docs/PUBLISHING.md` follows the same five-section checklist
  structure used in `junkerderprovinz/krusader`.
- Banner palette uses a neutral dark colour set
  (`#232629` / `#3daee9` / `#bdc3c7` / `#fcfcfc`) purely for GitHub
  README presentation. This is **not** a product theme — Standard Notes
  has no relationship to Breeze Dark or any KDE styling. Treat the
  palette as repo-decorative only.

## Things the user may want to revisit before publishing

1. Open an Unraid forum support thread and replace the GitHub-Issues
   `<Support>` URL in both templates with the new forum URL — see
   `docs/PUBLISHING.md` § 3.
2. After making the GitHub repo public, fetch each `<TemplateURL>` and
   `<Icon>` URL once with `curl -fsI` to confirm `200 OK` — see
   `docs/PUBLISHING.md` § 2.
3. Tag a `v0.1.0` release once the templates have been smoke-tested on
   a real Unraid host.
4. Consider rasterising `banner.svg` to a 1280x320 PNG and committing
   it alongside, in case any downstream consumer doesn't render SVG.
5. Submit to the Unraid Community Applications feed
   (`Squidly271/AppFeed`) — full procedure in `docs/PUBLISHING.md`
   § 4.

## Not done (out of scope per instructions)

- No `git remote add` / `git push`.
- No `gh repo create`.
- No registration with the Unraid Community Applications feed.
- No Unraid forum thread created (cannot be done from here).
