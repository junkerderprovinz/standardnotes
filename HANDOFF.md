# HANDOFF — standardnotes-server (Unraid templates)

Local working directory. **Publishing remains paused** — nothing has
been pushed, no remote has been created, and no Community Applications
submission has been made. Do not push, create a repo, or publish until
the user explicitly approves.

## Latest pass — server/webui repo split, container rename

The repo is being split into two siblings:

- **`junkerderprovinz/standardnotes-server`** — this directory.
  Holds the backend template (`standardnotes/server` image,
  `StandardNotes-LocalStack` companion). Container name in the
  template is now **`StandardNotesServer`** (was `StandardNotes`)
  so the name is unambiguous against the WebUI container.
- **`junkerderprovinz/standardnotes-webui`** — new sibling repo
  with its own working directory at
  `/home/user/workspace/standardnotes-webui/`. Ships the official
  `standardnotes/web` browser client as an Unraid template,
  container name **`StandardNotes`** (the user-facing brand
  name now belongs to the web container).

What changed in this directory:

- **Repo URL switch.** Every `<TemplateURL>`, `<Icon>`, `<Support>`,
  badge URL, install command, and PUBLISHING.md raw-URL example moved
  from `junkerderprovinz/standardnotes` to
  `junkerderprovinz/standardnotes-server`. Raw URLs now resolve under
  `https://raw.githubusercontent.com/junkerderprovinz/standardnotes-server/main/...`.
  XML template *filenames* are unchanged (`standardnotes-server.xml`,
  `standardnotes-localstack.xml`).
- **Container rename: `StandardNotes` → `StandardNotesServer`.**
  Updated in `templates/standardnotes-server.xml` (`<Name>`),
  README §10 update commands (`docker stop StandardNotesServer && docker
  rm StandardNotesServer`), README §2 architecture diagram, README §8
  NPM forward-host wording, README §11 `nc -zv` snippet, and
  `docs/sync-loop-troubleshooting.md` `docker exec`/`docker logs`
  examples. The `Target=` env-var keys are unchanged so the running
  container env is identical.
- **WebUI cross-references.** README §1 "What is this?" and §1 "Scope"
  bullets now point users at the companion repo
  `junkerderprovinz/standardnotes-webui` (container `StandardNotes`,
  default host port `3001` to avoid the backend's `:3000`). The server
  XML `<Overview>` notes the same — that the web template is in the
  separate companion repo and uses container name `StandardNotes`.
- **LICENSE credit** updated from
  `standardnotes-unraid contributors` →
  `standardnotes-server contributors` to match the new repo name.
- **All previous decisions kept.** MariaDB-only wording, IP-based DB /
  Redis hosts (`192.168.x.x`), no Redis password, secrets masked,
  `COOKIE_DOMAIN` and `PUBLIC_FILES_SERVER_URL` visible, files-server
  port placement under Public Files Server URL, `<Support>` pointing
  at GitHub Issues until a forum thread exists.

### Files changed in this pass

```
standardnotes-unraid/   (working dir; published as standardnotes-server)
├── README.md                              # repo URLs, container
│                                          # rename, WebUI pointer
├── HANDOFF.md                             # this update
├── LICENSE                                # credit line rename
├── docs/
│   ├── PUBLISHING.md                      # repo URLs
│   └── sync-loop-troubleshooting.md       # container name in
│                                          # docker exec / logs
└── templates/
    ├── standardnotes-server.xml           # <Name>StandardNotesServer</Name>,
    │                                      # repo URLs, WebUI mention,
    │                                      # files-port description IP
    │                                      # wording
    └── standardnotes-localstack.xml       # repo URLs
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

$ grep -rnE "junkerderprovinz/standardnotes($|[^-])" \
    README.md templates/ docs/PUBLISHING.md \
    docs/configuration.md docs/sync-loop-troubleshooting.md \
    examples/ .github/workflows/
(no matches — all user-facing URLs now carry the -server suffix.
 HANDOFF.md retains historical references for context.)

$ grep -rnE "<Name>StandardNotes</Name>" templates/
(no matches — server container is now StandardNotesServer; the
 plain "StandardNotes" name belongs to the WebUI repo.)
```

---

## Earlier pass — wordmark-free icon, paid-features / WebUI / AiO scope note

Per direct user request (in German):

1. *"Als Container logo bitte Das logo ohne schrift aus dem Banner
   verwenden ohne Schriftzug."* — use the mark from the banner, no
   wordmark.
2. Whether self-hosting unlocks paid Standard Notes styling / features.
3. Whether the template should ship a Web UI bundled in, separately,
   or as an AiO container.

What changed:

- **`.github/assets/icon.svg` rebuilt around the official mark only.**
  The icon previously was a generic dark "note + lock" illustration.
  It is now the same 188×188 brand-blue (`#1C6EE0`) mark used in
  `.github/assets/banner.svg`, centered in a 256×256 transparent
  square (34px padding all sides). No "Standard Notes" wordmark, no
  Proton / powered-by markings in the visual. `aria-label="Standard
  Notes"` is the only textual reference and is non-visible
  accessibility metadata.
- **Icon URLs verified, no template change needed.** Both
  `templates/standardnotes-server.xml` and
  `templates/standardnotes-localstack.xml` already point `<Icon>` at
  `https://raw.githubusercontent.com/junkerderprovinz/standardnotes/main/.github/assets/icon.svg`,
  which is the file we updated, so the new icon will be picked up
  automatically once the repo is published.
- **README §1 — new "Scope: backend only, no paid-feature unlocks,
  no AiO image" subsection.** Three bullets, written in plain
  English, that directly answer the user's three questions:
  - Paid Standard Notes features are gated by upstream's licensing /
    subscription checks and are **not** unlocked by self-hosting;
    this template deploys the upstream image as-is and does not
    patch those checks.
  - The web UI is intentionally a *separate* optional container
    (official `standardnotes/web`), not bundled into this template.
    Reasoning called out: mirrors upstream's
    `docker-compose.example.yml`, independent upgrades, no custom
    rebuilt AiO image to maintain.
  - No All-in-One container is planned for the first Community
    Applications release. Justified: would require a custom rebuilt
    image diverging from upstream tags, would re-bundle MariaDB /
    Redis most users already run, and would need re-release on every
    upstream component bump.
  - A dedicated `StandardNotes-Web` Community template is **not**
    shipped in this pass. The current upstream env-var surface
    (default sync-server URL, listening port) of the official web
    image needs verification against the live tag before publishing —
    a wrong default here would silently point users at the public
    Standard Notes sync server instead of their self-hosted backend.
    Documented as a known gap so the next pass can verify and add
    the template safely.
- **Validation.**
  - `python3 -m xml.etree.ElementTree` parses
    `.github/assets/icon.svg`, `.github/assets/banner.svg`,
    `templates/standardnotes-server.xml`, and
    `templates/standardnotes-localstack.xml` cleanly.
  - `grep -rn "[Pp]roton\|powered.?by"` against
    `.github/assets/` returns zero hits. The banner still contains
    its intended "Standard Notes" wordmark; the icon has only an
    `aria-label` (non-visible).
  - No real domains, no `bottich`, no `192.168.20.*` introduced.
- **No push, no remote action.** Working tree only — main agent will
  review and push.

## Earlier pass — generic examples, visible URL/Cookie fields, files-port relocation

Per direct user request. Scope kept small.

- **No real domain anywhere in templates/docs/examples.** Replaced
  `standardnotes.bottich.lol` / `files.standardnotes.bottich.lol`
  with generic `standardnotes.mydomain.tld` /
  `files.standardnotes.mydomain.tld` across `README.md`,
  `templates/standardnotes-server.xml`, `examples/.env.example`, and
  the `docs/` files. The "test deployment" sample table in the
  README is now a generic shape reference (no real LAN IPs, no real
  user/db names).
- **`192.168.x.x` placeholders.** Every host example for
  `DB_HOST` / `REDIS_HOST` / NPM / SWAG / nginx upstream now uses
  `192.168.x.x` (no real `192.168.20.*` addresses).
- **XML — concise required wording per user request.**
  - `MariaDB Host` description: *"IP address of your MariaDB
    container."* + default `192.168.x.x`.
  - `MariaDB Password` description: *"Password for the MariaDB user
    above."*.
  - `Redis Host` description: *"IP address of your Redis
    container."* + default `192.168.x.x`.
- **Visible (not hidden/advanced) fields.** `PUBLIC_FILES_SERVER_URL`
  and `COOKIE_DOMAIN` are now `Display="always"` so users see them
  on the main template page. They remain `Required="false"`.
- **Files Server Port repositioned + clearer NPM steps.** The
  `Files Server Port` `<Config>` is now placed directly under
  `Public Files Server URL` in the XML (moved out of the PORTS
  block at the top). Its description spells out the second-NPM-host
  recipe: Domain `files.standardnotes.mydomain.tld`, Scheme http,
  Forward IP = StandardNotes container IP, Forward Port `3125`,
  Websockets ON, Let's Encrypt + Force SSL + HTTP/2 ON.
- **`COOKIE_DOMAIN` doubles as the public sync domain note.** Its
  description now states explicitly that clients enter
  `https://standardnotes.mydomain.tld` as their Custom Sync Server
  and that **no separate "container domain" env var is required**
  for this template (the upstream `standardnotes/server` image does
  not document one — only `COOKIE_DOMAIN` and
  `PUBLIC_FILES_SERVER_URL` are exposed). README's `COOKIE_DOMAIN`
  table row reflects this.
- **Validation.**
  - `python3 -m xml.etree.ElementTree` parses
    `templates/standardnotes-server.xml` cleanly.
  - `<Config>` order verified: API/Gateway Port → Paths → MariaDB
    block → Redis block → Secrets → Public Files Server URL →
    Files Server Port → Cookie Domain.
  - `grep -rn "bottich\|192\.168\.20\."` against `templates/`,
    `docs/`, `README.md`, `examples/` returns no hits. (HANDOFF.md
    intentionally still references the old values as historical
    context for prior passes.)

## Latest pass — UI polish, IP-only host wording, README badges

This pass tightens the Unraid UI presentation, clarifies the
host-address model, answers the "second subdomain required?" question
explicitly, and adds a polished badge row to the README.

- **Container name is now exactly `StandardNotes`** in
  `templates/standardnotes-server.xml` (`<Name>StandardNotes</Name>`,
  per the user's request). README §10 update commands and
  `docs/sync-loop-troubleshooting.md` `docker exec` / `docker logs`
  examples updated to match. The XML *filename* stays
  `standardnotes-server.xml`.
- **Variable display names → nice labels.** The `<Config Name="…">`
  for every variable is now a human label (`MariaDB Host`,
  `MariaDB Port`, `MariaDB User`, `MariaDB Password`,
  `MariaDB Database Name`, `Database Driver`, `Redis Host`,
  `Redis Port`, `Cache Backend`, `JWT Secret`,
  `Auth Server Encryption Key`, `Valet Token Secret`,
  `Public Files Server URL`, `Cookie Domain`). The underlying
  `Target=` env-key values (`DB_HOST`, `DB_USERNAME`, …) are
  unchanged so the running container env is identical.
- **IP-based host wording, container-name preference removed.** The
  template's `DB_HOST` / `REDIS_HOST` descriptions now say:
  "IP address of your … container, e.g. 192.168.20.71 / .72 — IP is
  preferred — unambiguous and stable." The README §0 cause list,
  §3 example table, §5 Redis section, §6 configuration table, §11
  troubleshooting entry, `docs/configuration.md`, and
  `docs/sync-loop-troubleshooting.md` are all updated to match. The
  README §3 lead-in and the example NPM nginx snippet now point at
  `192.168.20.38` rather than a `standardnotes-server` container
  name.
- **Compact variable descriptions** with explicit secret-generation
  hints. `JWT Secret`, `Auth Server Encryption Key`,
  `Valet Token Secret` each carry `Generate with: openssl rand -hex 32`
  in the description. `MariaDB Password` description reminds the user
  that they should use the password they set when creating the DB
  user (no need to generate a fresh secret unless they're also
  creating the user). `Cookie Domain` and `Public Files Server URL`
  descriptions reduced to single tight paragraphs.
- **Redis password not added.** The official `standardnotes/server`
  image does not document a Redis-auth env var. Adding one without
  upstream confirmation would risk broken installs. Instead the
  Redis Host description and the README §5 / `docs/configuration.md`
  carry a compact security note: keep Redis on a trusted VLAN /
  private bridge and firewall it off the public internet; only add
  auth if your specific image fork documents it.
- **README §8 — second subdomain question answered explicitly.**
  Section retitled "Files server — is a second subdomain required?"
  with a "Short answer: No, but recommended for fully configured
  attachment support" lead. Re-states upstream's two-port,
  two-URL split (sync `:3000`, files `:3125` +
  `PUBLIC_FILES_SERVER_URL`). Same-domain path proxying called out
  as **not recommended for this community template** rather than
  "not reliably supported" — clearer wording, same outcome.
- **README badge row** rebuilt as a polished shields.io row near the
  top: Unraid Community Template, Standard Notes self-hosted,
  Docker `standardnotes/server`, MariaDB, Redis, GitHub Actions
  validate workflow, License (MIT). All badges link to relevant
  sections / repos and use brand colours where applicable.
- **`examples/.env.example`** — `DB_HOST` and `REDIS_HOST` switched
  to IP-style examples (`192.168.20.71`, `192.168.20.72`); Redis
  block notes the absence of a password env and the VLAN /
  firewall recommendation.

### Files changed in this pass

```
standardnotes/
├── README.md                              # badge row, container
│                                          # name updates, IP-only
│                                          # host wording, §3
│                                          # example table, §5 Redis
│                                          # security note, §8
│                                          # second-subdomain answer
├── HANDOFF.md                             # this update
├── docs/
│   ├── configuration.md                   # IP wording, Redis
│   │                                      # security note
│   └── sync-loop-troubleshooting.md       # IP wording, container
│                                          # name in docker exec /
│                                          # docker logs
├── templates/
│   └── standardnotes-server.xml           # <Name>StandardNotes</Name>,
│                                          # nice variable labels,
│                                          # IP-based host
│                                          # descriptions, compact
│                                          # secret-gen hints
└── examples/
    └── .env.example                       # IP-based DB / Redis
                                           # examples, Redis security
                                           # note
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

$ grep -niE "container.name|by name" README.md templates/*.xml \
    docs/*.md examples/.env.example
templates/standardnotes-localstack.xml:26:      Unraid's container-name DNS (…)
# (only match — and that one is the LocalStack INTERNAL hostname
#  the upstream image expects on the docker bridge, not a
#  user-facing host field. Intentionally retained.)

$ grep -nE "REPLACE_WITH_[A-Z_]+" templates README.md docs examples
(no matches — clean)
```

---



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

## Latest pass — shared banner & icon across server/webui

`.github/assets/banner.svg` and `.github/assets/icon.svg` are now byte-
identical across the `standardnotes-server` and `standardnotes-webui`
repos. The banner is the clean Standard Notes lockup (white background,
official mark + "Standard Notes" wordmark, no taglines, no Proton or
powered-by text). The icon is the bare Standard Notes mark on a 256×256
canvas, suitable as the Docker/Unraid container icon. Each template's
`<Icon>` element still points at its own repo's raw URL
(`standardnotes-server` or `standardnotes-webui`) but they reference the
same relative asset path `.github/assets/icon.svg`, so future asset
changes only need to be replicated across both repos.

## Not done (out of scope per instructions)

- No `git remote add` / `git push`.
- No `gh repo create`.
- No registration with the Unraid Community Applications feed.
- No Unraid forum thread created (cannot be done from here).
