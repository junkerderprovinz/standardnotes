# HANDOFF — standardnotes-server (Unraid templates)

Local working directory. **Publishing remains paused** — nothing has
been pushed, no remote has been created, and no Community Applications
submission has been made. Do not push, create a repo, or publish until
the user explicitly approves.

## Latest pass — PNG icon, concise template descriptions, live-install success note

User reported:

- Unraid does not render SVG container icons. Templates need a PNG.
- The server template's `<Overview>` and several `<Config Description>`
  fields contained typographic em dashes and Markdown-style link
  syntax (`[https://...](https://...)`) that rendered as visible literal
  characters in the Unraid UI.
- The LocalStack template's variable descriptions were over-long for
  the Unraid sidebar.
- A live install on a `br0.20` VLAN with the server pointed at the
  user's actual Redis IP, plus the bootstrapped LocalStack reachable as
  `localstack`, now syncs cleanly. Note duplication stopped.

### What changed in this pass

- **`.github/assets/icon.png`** (new). 256×256 RGBA PNG rasterised from
  the existing wordmark-free `.github/assets/icon.svg` via ImageMagick
  (`magick -background none -density 384 icon.svg -resize 256x256
  PNG32:icon.png`). Transparent canvas, brand-blue (`#1C6EE0`) mark.
  Byte-identical to the PNG shipped in the companion `standardnotes-webui`
  repo. SVG retained for source.
- **`templates/standardnotes-server.xml`**:
  - `<Icon>` now points at `.github/assets/icon.png` instead of the SVG.
  - `<Overview>` rewritten end-to-end. All em dashes (`—`) replaced
    with plain hyphens or removed. All Markdown-style autolinks like
    `[https://...](https://...)` removed. Prose condensed to short
    sentences and lists that render cleanly in the Unraid UI. Kept the
    decision-relevant facts: required side containers, port mapping
    (`3125` host → `3104` container), URL shapes for `COOKIE_DOMAIN` /
    `PUBLIC_FILES_SERVER_URL` / Custom Sync Server, hostname
    `localstack` resolution recipes, and the sync-loop warning.
  - `<Config>` descriptions (API port, Logs/Uploads, MariaDB block,
    Redis block, secrets, Public Files Server URL, Files Server Port,
    Cookie Domain): em dashes replaced with hyphens or commas. Each
    description trimmed to the essentials; the long commentary moved
    out of the template into the README where it already lives.
- **`templates/standardnotes-localstack.xml`**:
  - `<Icon>` now points at the PNG.
  - `<Overview>` rewritten in the same style as the server template -
    short paragraphs, hyphens instead of em dashes, no Markdown link
    syntax. Kept the hostname-resolution recipes and the bootstrap
    `curl`/`chmod` + `awslocal` verify commands.
  - `<Config>` descriptions reduced to user-requested essentials:
    - `SERVICES`: `Required services. Keep sns,sqs.`
    - `HOSTNAME_EXTERNAL`: `Keep localstack. Server resolves this name.`
    - `LS_LOG`: `Log level. Default warn.`
    - `LocalStack Bootstrap Script`: `Required bootstrap script.
      Creates Standard Notes SNS topics and SQS queues. See Overview
      for the curl + chmod host-prep commands.`
- **`HANDOFF.md`** (this entry). Live-install success: after pointing
  Redis at the correct VLAN IP and finishing the LocalStack bootstrap
  + `--add-host=localstack:<LocalStack-IP>` recipe, sync round-trips
  cleanly and note duplication has stopped. No public README rewrite
  for the success story since it is operational, not a code change.

### Preserved across this pass

- `COOKIE_DOMAIN` is a bare domain, `PUBLIC_FILES_SERVER_URL` is a
  full HTTPS URL.
- Files port mapping host `3125` → container `3104`.
- MariaDB-only wording (`DB_TYPE=mysql` is the internal driver value).
- Container names `StandardNotesServer`, `StandardNotes-LocalStack`,
  `StandardNotes` (in the webui repo).
- All krusader-style inner-text values on every non-empty
  `Default=`.
- All previous README content (sync-loop guardrails, troubleshooting,
  configuration tables) untouched.

### Validation summary for this pass

- `magick identify` and `file` on `.github/assets/icon.png`: PNG, 256×256,
  RGBA, non-empty. Both repos' PNGs have identical md5.
- `python3 -c "import xml.etree.ElementTree as ET; ET.parse(p)"` on both
  `templates/*.xml` and `.github/assets/icon.svg`: all OK.
- `python3 -c "import yaml; yaml.safe_load(open(p))"` on
  `.github/workflows/*.yml`: OK.
- `grep -nE "—|\[https://" templates/*.xml`: no matches in either
  template (em dashes and Markdown link syntax fully removed from
  descriptions).

## Latest pass — dockerMan inner-text audit + edit-screen "reset" troubleshooting

User reported that on Unraid, opening *Edit* on a freshly installed
StandardNotesServer container shows the template defaults rather than
the values that were saved at install time. Comparison with the
user's working krusader template (`<Config ... Default="3000">3000</Config>`)
suggested dockerMan persists / re-displays the inner text node of each
`<Config>` element, not just the `Default=` attribute.

Audit result: every `<Config>` in this repo's templates that has a
non-empty `Default=` **already** carries matching inner text (krusader
style). The self-closing `<Config .../>` elements that remain all have
`Default=""` and are either:

- Sensitive blanks (passwords / secrets) — `DB_PASSWORD`,
  `AUTH_JWT_SECRET`, `AUTH_SERVER_ENCRYPTION_SERVER_KEY`,
  `VALET_TOKEN_SECRET`. These intentionally have no default, so an
  empty/self-closing element is correct.
- Optional blanks — `PUBLIC_FILES_SERVER_URL`, `COOKIE_DOMAIN`. Same
  reasoning: empty default, no user-real-domain shipped in the
  template.

No template XML mutation was needed; the audit is recorded here so
the next person doesn't redo it. The same holds for
`templates/standardnotes-localstack.xml` (all non-empty defaults
already have matching inner text) and the companion webui repo's
`templates/standardnotes-webui.xml`.

### What changed in this pass

- **`README.md`** § 11 *Troubleshooting* — new collapsible entry
  *"Unraid edit screen shows template defaults / values look reset"*.
  Explains the two ways users hit it: (a) opening the template from
  *Apps* / *Add Container* / *Templates* (which always shows
  defaults) instead of clicking the container icon → *Edit* (which
  loads the live, saved values), and (b) overwriting the saved
  per-container `my-*.xml` (`my-StandardNotesServer.xml`,
  `my-StandardNotes-LocalStack.xml`, `my-StandardNotes.xml`) with
  `curl` after configuring. Also notes that the templates already
  ship inner text matching every non-empty `Default=`, so dockerMan
  persists and re-displays each value reliably.
- **`HANDOFF.md`** (this section).

Files changed in this pass:

- `README.md`
- `HANDOFF.md`

No template XML changed; audit confirmed templates are already
krusader-style.

Validation summary for this pass:

- `python3 -c 'import xml.etree.ElementTree as ET; ET.parse(p)'` on
  both `templates/*.xml` and `.github/assets/*.svg` → all OK.
- `python3 -c 'import yaml; yaml.safe_load(open(p))'` on
  `.github/workflows/*.yml` → OK.
- Spot-check: every `Default="<non-empty>"` in
  `templates/standardnotes-server.xml` and
  `templates/standardnotes-localstack.xml` has matching inner text.
  Every remaining self-closing `<Config .../>` has `Default=""`.

## Earlier pass — static-IP / VLAN install path, redis:7-alpine disambiguation

User clarified two things from a live `br0.20` VLAN install where each
container has its own fixed IP:

1. They already run an **official Redis container** for the sync
   cache. The `redis:7-alpine` references in earlier docs caused
   confusion — they were never meant to be a second Redis server,
   only a disposable client container to get `redis-cli` on demand.
   The throwaway `docker run --rm redis:7-alpine redis-cli ... ping`
   they tried timed out because it ran from the default Docker
   bridge, which on a `br0.20` VLAN install cannot route to the
   VLAN Redis container.
2. The user has no LocalStack container running yet:
   `docker exec StandardNotesServer getent hosts localstack` is
   empty and a TCP probe to `localstack:4566` returns
   `getaddrinfo ENOTFOUND localstack`. For their VLAN / fixed-IP
   setup, the working install recipe is: install
   `StandardNotes-LocalStack` on the **same `br0.<vlan>`** with its
   own **fixed IP** (e.g. `192.168.x.x`), then **append**
   `--add-host=localstack:<LocalStack-IP>` to `StandardNotesServer`'s
   *Extra Parameters* and restart the server container.

This pass rewords the Redis-test guidance everywhere so the
authoritative test is the **in-container `node -e` TCP probe**, and
the disposable `redis:7-alpine redis-cli` form is presented only as
a convenience that **requires `--network`** on `br0` / macvlan / VLAN
/ static-IP setups. The LocalStack install path is spelled out for
the static-IP case as: install on same VLAN + fixed IP + `--add-host`
on the server.

### What changed in this pass

- **`README.md`**:
  - § 0 *Emergency checklist* step 4 (Redis): rewritten. Primary test
    is the in-container `node -e` probe; the `docker run --rm
    redis:7-alpine redis-cli ... ping` form is moved to a callout
    note that explicitly says `redis:7-alpine` is a disposable
    client container (not a second Redis server) and that it
    **requires `--network <same-network-as-redis>`** on `br0` /
    macvlan / VLAN / static-IP installs, otherwise the default
    bridge cannot route to the VLAN Redis and the test times out
    even when Redis is healthy.
  - § 0 *Emergency checklist* step 5 (LocalStack) callout box:
    static-IP / `br0` / macvlan note rewritten to spell out the
    install recipe — same-VLAN container + fixed IP +
    `--add-host=localstack:<LocalStack-IP>` on
    StandardNotesServer's *Extra Parameters* + restart.
  - § 3 *Step 2 — Start LocalStack* now has explicit per-network-type
    install instructions: user-defined Docker network path
    (alias `localstack`) and the `br0` / macvlan / VLAN / static-IP
    path (fixed IP on same VLAN + `--add-host` on the server +
    restart). The verification callout now points out that an empty
    `getent hosts localstack` on a static-IP install almost always
    means the `--add-host` line is missing.
  - § 11 *Notes are being duplicated* duplicates checklist: the
    Redis verification step now uses the in-container `node -e`
    probe as the authoritative test and notes that the
    `redis:7-alpine` throwaway is only valid with `--network` and
    is otherwise misleading on VLAN setups.
  - § 11 *SNS / SQS errors* `<details>` block: empty-`getent`
    recovery now names the static-IP / VLAN install recipe (same
    VLAN + fixed IP + `--add-host` + restart), not just the bare
    `--add-host` flag.

- **`docs/sync-loop-troubleshooting.md`**:
  - § Emergency checklist step 4 (Redis) and § 2 *Redis reachability*
    rewritten to lead with the in-container `node -e` probe as the
    authoritative test. The `redis:7-alpine` disposable-client form
    is kept as an optional CLI alternative with explicit `--network`
    caveats and a "not a second Redis server" disclaimer.
  - Static-IP / `br0` / macvlan / VLAN callout in both the
    emergency checklist (step 5) and § 2a *LocalStack reachability*
    rewritten to spell out the install recipe — same VLAN, fixed
    IP, `--add-host` on the server, restart — instead of just
    saying "add `--add-host`".

- **`templates/standardnotes-server.xml`**:
  - `<Overview>` *LOCALSTACK / ENOTFOUND localstack WARNING* block
    rewritten with two labelled fix paths: (A) user-defined Docker
    network + alias, and (B) `br0` / macvlan / VLAN / static-IP
    install — same VLAN + fixed IP + `--add-host` appended to
    *Extra Parameters* + restart. Added a Redis-test note that
    `docker run --rm redis:7-alpine redis-cli ... ping` from the
    default bridge is **not** a reliable test on static-IP setups
    and that the authoritative test runs inside this container.
  - `<Requires>` extended to call out the static-IP / VLAN install
    recipe with the `--add-host` line.
  - *Redis Port* Config description rewritten — authoritative test
    is the in-container `node -e` probe; `redis:7-alpine` is named
    as a disposable redis-cli client (not a second Redis server)
    and the default-bridge / VLAN routing caveat is stated.

- **`templates/standardnotes-localstack.xml`**:
  - `<Overview>` *HOSTNAME RESOLUTION* block rewritten with two
    explicitly labelled fix paths (user-defined Docker network vs.
    br0 / macvlan / VLAN / static-IP install with same VLAN + fixed
    IP + `--add-host` on the server + restart).
  - `HOSTNAME_EXTERNAL` Config description updated to match the
    new fix recipe.

### Preserved across this pass

- `COOKIE_DOMAIN` = bare domain only (no `https://`), distinct from
  the client's *Custom Sync Server* URL which is a full HTTPS URL.
- `PUBLIC_FILES_SERVER_URL` = full HTTPS URL (with `https://`).
- Files port mapping: host `3125` → container `3104`.
- MariaDB-only (`DB_TYPE=mysql` is the internal driver value).
- Container name `StandardNotesServer`; LocalStack companion
  `StandardNotes-LocalStack`; LocalStack alias / hostname
  `localstack`.
- Generic example domains (`standardnotesserver.mydomain.tld`,
  `files.standardnotesserver.mydomain.tld`).

### Validation

- `python3 -c "xml.etree.ElementTree.parse(...)"` — both
  `templates/*.xml` parse cleanly.
- Same parse on `.github/assets/banner.svg` and
  `.github/assets/icon.svg` — both OK.
- `python3 yaml.safe_load_all` on
  `.github/workflows/{lint,build}.yml` — both OK.
- No pushes performed.

## Latest pass — LocalStack required, Redis/LocalStack reachability

User reported live-install debug findings:

- Redis test from a throwaway Docker container timed out: `Could not
  connect to Redis at 192.168.20.72:6379: Operation timed out`.
- Server worker logs show repeated `SQSError: SQS receive message
  failed: getaddrinfo ENOTFOUND localstack` in
  `/var/lib/server/logs/files-worker.log`.
- Missing / unresolvable LocalStack plausibly destabilises the worker
  pipeline and contributes to sync / duplication issues.

This pass reclassifies LocalStack from *optional* to *required
companion for the official Standard Notes server image* across all
user-facing surfaces, and adds first-class Redis / LocalStack
reachability checks (with the exact `redis-cli` and `node -e` TCP /
DNS commands) to the emergency duplicate-loop checklist.

### What changed in this pass

- **`README.md`**:
  - § 0 *Common causes* list: Redis bullet rewritten to mention
    `ECONNREFUSED` *and* connection timeouts (firewall / VLAN /
    `br0`); new LocalStack bullet describing the `ENOTFOUND
    localstack` failure mode and pointing to § 2.
  - § 0 *Emergency checklist*: expanded from 6 to 7 steps. Step 2
    now leaves LocalStack running alongside MariaDB / Redis. New
    step 4 = explicit Redis reachability checks (`docker run --rm
    redis:7-alpine redis-cli -h ... ping`, in-container `node -e`
    TCP probe). New step 5 = explicit LocalStack reachability
    checks (`getent hosts localstack`, in-container `node -e` TCP
    probe to `localstack:4566`), plus a *Static IP / `br0` /
    macvlan* note covering the two working fixes (user-defined
    Docker network + alias, or `--add-host=localstack:<IP>`).
  - § 1 *What is this?*: "optional **LocalStack** companion" →
    "**LocalStack** companion template (required companion for the
    official Standard Notes server image)". Per-concern bullet and
    comparison table updated to match. "Scope: backend only"
    bullet "plus the optional LocalStack companion" → "plus the
    required LocalStack companion".
  - § 2 *Architecture* bullet for LocalStack: marked as **required
    companion** with the `getaddrinfo ENOTFOUND localstack` failure
    mode spelled out.
  - § 3 *Pre-flight*: new bullet — LocalStack container resolvable
    as `localstack`. § 3 templates list: LocalStack template
    described as "required companion SNS / SQS provider".
  - § 3 *Step 2* heading: "(optional but recommended)" → "(required
    companion)". Added a callout box with the exact `getent hosts
    localstack` + `node -e` TCP probe commands and a pointer to
    § 11 / `docs/sync-loop-troubleshooting.md` for DNS fix-ups.
  - § 11 *SNS / SQS errors in the log* `<details>` block rewritten:
    title now includes the exact `getaddrinfo ENOTFOUND localstack`
    string; body explains LocalStack is required, includes the
    `getent hosts localstack` + `node -e` probe, and the
    user-defined-network-alias / `--add-host` fix recipe. Files-only
    nuance retained.
  - § 11 *Notes are being duplicated* `<details>` block: log-grep
    list extended with the `ENOTFOUND localstack` line; verification
    block now includes the `redis-cli` and `node -e` TCP probes for
    Redis *and* the `getent hosts localstack` probe.

- **`templates/standardnotes-server.xml`**:
  - `<Overview>` *Required, in this order* list: item 3 (LocalStack)
    is no longer marked *Optional* — it is described as the required
    companion, with the `ENOTFOUND localstack` failure mode named.
  - `<Overview>` new "LOCALSTACK / ENOTFOUND localstack WARNING"
    block: tells users to install / start LocalStack and verify
    `docker exec StandardNotesServer getent hosts localstack`
    returns a non-empty line *before* testing notes. Includes both
    fix recipes (user-defined Docker network + alias, or
    `--add-host`).
  - `<Overview>` sync-loop warning extended to list LocalStack
    alongside Redis / `COOKIE_DOMAIN` / proxy / image tag.
  - `<Requires>` updated to call out the LocalStack companion and
    the `ENOTFOUND localstack` symptom directly.

- **`templates/standardnotes-localstack.xml`**:
  - Container name kept as `StandardNotes-LocalStack` to avoid
    disrupting users mid-setup; instead the `<Overview>` is rewritten
    to make the hostname-resolution requirement very explicit.
  - New "REQUIRED companion" paragraph naming the `ENOTFOUND
    localstack` failure mode in `files-worker.log`.
  - New "HOSTNAME RESOLUTION — THE SERVER MUST RESOLVE `localstack`"
    block spelling out the two working setups (user-defined Docker
    network + alias `localstack`, or `--add-host=localstack:<IP>`)
    and the `docker exec StandardNotesServer getent hosts
    localstack` verification step.
  - `HOSTNAME_EXTERNAL` description: clarifies that
    `HOSTNAME_EXTERNAL=localstack` alone is **not** what makes the
    server resolve the name — the resolution has to be wired up via
    a shared user-defined network alias *or* `--add-host`. No new
    env vars invented.
  - `--hostname=localstack` in `<ExtraParams>` retained from the
    earlier pass.

- **`docs/configuration.md`**:
  - New top-level section **LocalStack (SNS / SQS)** between
    *Cache* and *Secrets*. Calls LocalStack the "required companion
    for the official Standard Notes server image". Describes the
    `ENOTFOUND localstack` failure mode, the two resolution fixes
    (user-defined network + alias, or `--add-host`), and the exact
    `getent hosts localstack` + `node -e` TCP-probe verification
    commands.

- **`docs/sync-loop-troubleshooting.md`**:
  - Emergency checklist: expanded from 6 to 7 steps to mirror the
    README. Step 2 leaves LocalStack running alongside MariaDB /
    Redis. New step 4 = Redis reachability with `docker run --rm
    redis:7-alpine ... ping` and the in-container `node -e` TCP
    probe. New step 5 = LocalStack reachability with `getent hosts
    localstack` and the in-container `node -e` probe to
    `localstack:4566`, plus the static-IP / `br0` / macvlan
    network-alias / `--add-host` recipe.
  - § 2 *Redis reachability*: expanded with the
    `docker run --rm redis:7-alpine redis-cli ... ping` throwaway
    test (works while the server is stopped) and the
    `docker exec StandardNotesServer node -e ...` in-container TCP
    probe. The existing `nc` snippet is retained as a fallback.
    Failure-mode paragraph now distinguishes refusal vs timeout.
  - New § 2a **LocalStack (SNS / SQS) reachability**: full
    checklist with the `getent hosts localstack` + `node -e`
    probes, the two working fixes (user-defined network alias vs
    `--add-host`), a `tail -n 200 .../files-worker.log` cleanup
    check, and the expected failure mode.
  - § 6 *Logs to grep for* table: added rows for Redis connection
    timeouts (`Operation timed out`) and for `SQSError: SQS receive
    message failed: getaddrinfo ENOTFOUND localstack` in
    `files-worker.log`.

### Decisions kept from earlier passes

- Container name `StandardNotesServer` (server XML).
- Container name `StandardNotes-LocalStack` retained — changing it
  to a bare `localstack` would disrupt existing users; instead the
  hostname-resolution requirement is documented explicitly.
- `--hostname=localstack` in `<ExtraParams>` of the LocalStack XML
  (sets the kernel hostname; resolution from the server container
  still requires shared network + alias or `--add-host`).
- MariaDB-only wording everywhere user-facing.
- All previous `https://` placement, files-port `3125:3104`, and
  generic-domain decisions.
- No unsupported env vars invented on the LocalStack template; the
  fix recipes use Docker-native network aliases or `--add-host`.

### Validation (this pass)

```
$ python3 -c "import xml.etree.ElementTree as ET
> for f in ['templates/standardnotes-server.xml',
>           'templates/standardnotes-localstack.xml',
>           '.github/assets/banner.svg',
>           '.github/assets/icon.svg']:
>     ET.parse(f); print('ok', f)"
ok templates/standardnotes-server.xml
ok templates/standardnotes-localstack.xml
ok .github/assets/banner.svg
ok .github/assets/icon.svg

$ python3 -c "import yaml,glob; \
    [yaml.safe_load(open(p)) for p in glob.glob('.github/workflows/*.yml')]"
(no output — all workflow YAMLs parse cleanly)

$ grep -nE "ENOTFOUND localstack" README.md docs/*.md templates/*.xml
README.md: present in § 0, § 2, § 11 SNS/SQS block, § 11 duplicate block
docs/configuration.md: present in LocalStack section
docs/sync-loop-troubleshooting.md: present in emergency checklist,
  § 2a, § 6 logs table
templates/standardnotes-server.xml: present in <Overview> warning and
  <Requires>
templates/standardnotes-localstack.xml: present in <Overview>
```

### Files changed in this pass

```
standardnotes-unraid/
├── README.md                                # § 0 causes + emergency
│                                            # checklist (Redis +
│                                            # LocalStack probes), § 1
│                                            # "required companion"
│                                            # wording, § 2 LocalStack
│                                            # required, § 3 pre-flight
│                                            # + Step 2 callout, § 11
│                                            # SNS/SQS block rewrite,
│                                            # duplicate-notes block
│                                            # extended
├── HANDOFF.md                               # this entry
├── docs/
│   ├── configuration.md                     # new LocalStack section
│   └── sync-loop-troubleshooting.md         # emergency checklist
│                                            # expanded, § 2 Redis
│                                            # probes added, new § 2a
│                                            # LocalStack section,
│                                            # § 6 logs table extended
└── templates/
    ├── standardnotes-server.xml             # <Overview> LocalStack
    │                                        # warning, <Requires>
    │                                        # updated
    └── standardnotes-localstack.xml         # <Overview> required +
                                             # hostname resolution
                                             # block, HOSTNAME_EXTERNAL
                                             # description tightened
```

### Publishing status

Still **paused**. No `git push`, no remote add, no CA submission. This
pass is local-only, ready for the user to review.

---

## Earlier pass — explicit `https://` placement, files port 3125 vs 3104

User reported a misconfiguration where `COOKIE_DOMAIN` had been set to a
full URL (`https://...` and a wrong domain), causing session breakage.
This pass documents — everywhere the user is likely to look — exactly
where `https://` belongs and where it does not, plus clarifies that the
template's *Files Server Port* maps host `3125` to container `3104`
(per upstream `docker-compose.example.yml`), so reverse proxies must
forward to host `3125`, not container `3104`.

### Settings shape — single source of truth

| Setting | Where it lives | Value type | Includes `https://`? | Example |
|---|---|---|---|---|
| `COOKIE_DOMAIN` | Server env / Unraid template | **Bare domain** | ❌ No | `standardnotesserver.mydomain.tld` |
| `PUBLIC_FILES_SERVER_URL` | Server env / Unraid template | **Full HTTPS URL** | ✅ Yes | `https://files.standardnotesserver.mydomain.tld` |
| Custom Sync Server | Standard Notes client / web app UI | **Full HTTPS URL** | ✅ Yes | `https://standardnotesserver.mydomain.tld` |

### What changed in this pass

- **`templates/standardnotes-server.xml`**:
  - `<Overview>` gained an explicit "HTTPS — where it belongs and where
    it doesn't" block (three-line matrix: bare domain for
    `COOKIE_DOMAIN`, full HTTPS URL for `PUBLIC_FILES_SERVER_URL`, full
    HTTPS URL for the client's Custom Sync Server).
  - `<Overview>` files-port wording rewritten: host `3125` → container
    `3104` (per upstream `docker-compose.example.yml`); `3104` is the
    *internal* upstream service port; `3125` is what your reverse proxy
    forwards to.
  - `Public Files Server URL` description: explicitly says "FULL HTTPS
    URL — INCLUDES the `https://` scheme".
  - `Cookie Domain` description: explicitly says "BARE DOMAIN ONLY —
    NO protocol, NO `https://`, NO trailing slash, NO path", with a
    correct/wrong example pair, and a note that `Custom Sync Server` in
    the client *is* a full HTTPS URL.
  - `Files Server Port` description: spells out that `Target=3104` is
    the container port, `Default=3125` is the host port (matches
    upstream `3125:3104` mapping), and that NPM / Unraid users must
    forward to **host** port `3125`, not `3104`.
- **`README.md`**:
  - § 0 gained a new "`https://` — where it belongs and where it does
    NOT" subsection with the three-row matrix above and a paragraph on
    the failure mode of `COOKIE_DOMAIN=https://...`.
  - § 0 gained a new "Emergency checklist — duplicates happening RIGHT
    NOW" with six numbered steps: stop clients, stop server, do not
    reconnect, check Redis, check `COOKIE_DOMAIN` / Custom Sync Server
    URL, delete/recreate test account if it duplicated.
  - § 3 example values table: split the cookie-domain row from the
    files-server URL row from the Custom Sync Server row, each
    annotated with bare-domain vs full-URL.
  - § 6 ports table: split into a 3-column table (Host / Container /
    Purpose) and added an inline note on `3125 → 3104` semantics.
  - § 6 env-var table: `COOKIE_DOMAIN` and `PUBLIC_FILES_SERVER_URL`
    descriptions tightened with explicit ✅ / ❌ examples.
  - § 8 NPM section: `COOKIE_DOMAIN` / `PUBLIC_FILES_SERVER_URL` /
    Custom Sync Server triplet called out side-by-side after the NPM
    setup steps.
  - § 8 files-server subsection: rewritten to make "forward to host
    port `3125`, not `3104`" the lead.
- **`docs/configuration.md`**:
  - Optional table now includes both `PUBLIC_FILES_SERVER_URL` (full
    HTTPS URL) and `COOKIE_DOMAIN` (bare domain), each with explicit
    correct/wrong example.
  - New "`https://` placement — quick reference" three-row table after
    the Optional table.
  - Files server Ports row gained the "forward to host port `3125`,
    not `3104`" note.
- **`docs/sync-loop-troubleshooting.md`**:
  - New top-level "Emergency checklist (duplicate-loop happening
    NOW)" section between the leading callout and § 0. Six numbered
    steps (stop clients, stop server, don't reconnect, check Redis,
    check `COOKIE_DOMAIN` shape & Custom Sync Server URL shape,
    delete/recreate test account).
  - § 4 `COOKIE_DOMAIN` checklist line replaced with the explicit
    three-row matrix (bare domain vs full HTTPS URL).
  - § 4 gained a new line on Files Server Port: forward to host
    `3125`, not `3104`, with the upstream `3125:3104` reference.
- **`examples/.env.example`**:
  - `PUBLIC_FILES_SERVER_URL` comment block: leads with "FULL HTTPS
    URL — INCLUDES `https://`"; reminder block on `3125:3104` and
    "forward to host 3125, NOT 3104".
  - `COOKIE_DOMAIN` comment block: leads with "BARE DOMAIN ONLY. No
    protocol, NO `https://`"; quick-reference triplet aligned in a
    `# COOKIE_DOMAIN = ... / # PUBLIC_FILES_SERVER_URL = ... /
    # Custom Sync Server = ...` block.

### Decisions kept from earlier passes

- Container name `StandardNotesServer` (server XML).
- MariaDB-only wording everywhere user-facing.
- Generic example domains `standardnotesserver.mydomain.tld` and
  `files.standardnotesserver.mydomain.tld`.
- `<Config>` ordering preserved (Public Files Server URL → Files
  Server Port → Cookie Domain).
- `Target=3104` / `Default=3125` mapping confirmed against
  `/home/user/workspace/standardnotes_docker-compose.example.yml`
  (`3125:3104` in `services.server.ports`). No XML port-target change
  was needed; only wording.

### Validation (this pass)

```
$ python3 -c "import xml.etree.ElementTree as ET
> for f in ['templates/standardnotes-server.xml',
>           'templates/standardnotes-localstack.xml',
>           '.github/assets/banner.svg',
>           '.github/assets/icon.svg']:
>     ET.parse(f); print('ok', f)"
ok templates/standardnotes-server.xml
ok templates/standardnotes-localstack.xml
ok .github/assets/banner.svg
ok .github/assets/icon.svg

$ grep -nE "COOKIE_DOMAIN\s*=\s*https" README.md examples/.env.example \
    docs/*.md templates/*.xml
README.md:108:If you accidentally set `COOKIE_DOMAIN=https://standardnotesserver.mydomain.tld`,
# (only match — the wrong-usage example in the new § 0 explainer.
#  No live values use https:// in COOKIE_DOMAIN.)
```

### Files changed in this pass

```
standardnotes-unraid/
├── README.md                                # § 0 https://-matrix + emergency
│                                            # checklist; § 3 example table split;
│                                            # § 6 ports table reshaped;
│                                            # § 6 env-var table tightened;
│                                            # § 8 NPM triplet + files-port lead
├── HANDOFF.md                               # this entry
├── docs/
│   ├── configuration.md                     # COOKIE_DOMAIN added, Optional
│   │                                        # table widened, https://-placement
│   │                                        # quick reference, files-port note
│   └── sync-loop-troubleshooting.md         # emergency checklist, § 4 matrix,
│                                            # § 4 files-port line
├── examples/
│   └── .env.example                         # full HTTPS URL vs bare domain
│                                            # leads, files-port reminder
└── templates/
    └── standardnotes-server.xml             # Overview HTTPS-placement block,
                                             # PUBLIC_FILES_SERVER_URL "full
                                             # HTTPS URL" wording,
                                             # COOKIE_DOMAIN "bare domain
                                             # only" wording, Files Server
                                             # Port 3125→3104 explanation
```

### Publishing status

Still **paused**. No `git push`, no remote add, no CA submission. This
pass is local-only, ready for the user to review.

---

## Earlier pass — heading, MIT, Lint/Build CI split

- **README heading** is now `Standard Notes Server for Unraid` (was
  `Standard Notes for Unraid`). The banner image's `alt` text is updated
  to match.
- **LICENSE simplified to plain MIT.** Copyright holder is
  `Junker der Provinz`. The bundled-software notice was removed from
  the LICENSE file; the README's License section keeps a one-paragraph
  pointer to the upstream component licenses (AGPL-3.0 / Apache-2.0 /
  GPL-2.0) without re-licensing them.
- **Workflows split.** `.github/workflows/validate.yml` removed and
  replaced with two focused workflows, both stdlib-only:
  - `lint.yml` (job name `Lint`) — placeholder guard
    (`REPLACE_WITH_*`) and a tiny YAML sanity check on
    `.github/workflows/*.yml` (no tabs, has `name:`, `on:`, `jobs:`).
  - `build.yml` (job name `Build`) — verifies required repo assets
    exist (`templates/*.xml`, `.github/assets/*.svg`, `README.md`,
    `LICENSE`) and parses every XML/SVG with `xml.etree.ElementTree`.
- **README badge row.** The single `validate` badge was replaced with
  separate `lint` and `build` shields.io badges, each pointing at its
  workflow on `main`. The MIT license badge is now a static
  shields.io badge (was the dynamic `github/license/...` one) so it
  works before the repo is public.
- **License section in README** condensed to a single paragraph (MIT
  for the wrapper, upstream licenses for the referenced images).

### Validation (this pass)

```
$ python3 - <<'PY'
import xml.etree.ElementTree as ET, glob
for p in sorted(set(glob.glob('templates/*.xml') + glob.glob('.github/assets/*.svg'))):
    ET.parse(p); print('ok', p)
PY
ok .github/assets/banner.svg
ok .github/assets/icon.svg
ok templates/standardnotes-localstack.xml
ok templates/standardnotes-server.xml

$ python3 -c "import yaml,glob; [yaml.safe_load(open(p)) for p in glob.glob('.github/workflows/*.yml')]"
(no output — all workflow YAMLs parse cleanly)

$ grep -rnE "REPLACE_WITH_[A-Z_]+" templates README.md docs
(no matches)
```

---

## Earlier pass — server/webui repo split, container rename

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
  with generic `standardnotesserver.mydomain.tld` /
  `files.standardnotesserver.mydomain.tld` across `README.md`,
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
  recipe: Domain `files.standardnotesserver.mydomain.tld`, Scheme http,
  Forward IP = StandardNotes container IP, Forward Port `3125`,
  Websockets ON, Let's Encrypt + Force SSL + HTTP/2 ON.
- **`COOKIE_DOMAIN` doubles as the public sync domain note.** Its
  description now states explicitly that clients enter
  `https://standardnotesserver.mydomain.tld` as their Custom Sync Server
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

## Latest pass — banner centering & domain example consistency

- `.github/assets/banner.svg` rebuilt with the "Standard Notes" wordmark
  expressed as embedded vector paths (Liberation Sans Bold @ 92 px,
  letter-spacing -2). Layout is mathematically centered on the 1200×360
  canvas: mark (180) + gap (32) + wordmark (~654) = ~866, with ~167 px
  side padding on each side. Removes the previous renderer-dependent
  text width. Banner is byte-identical across `standardnotes-server`
  and `standardnotes-webui`. Icon unchanged.
- All user-facing references to the sync/API host now use
  `standardnotesserver.mydomain.tld` (and
  `files.standardnotesserver.mydomain.tld` for the files server)
  across `README.md`, `examples/.env.example`, and
  `templates/standardnotes-server.xml`. `COOKIE_DOMAIN` and
  `PUBLIC_FILES_SERVER_URL` examples updated accordingly.

## Latest pass — LocalStack bootstrap script (SNS/SQS init)

User reported: LocalStack TCP `4566` reachable from
`StandardNotesServer` and Redis reachable, but **account creation
still hangs and the first note duplicates infinitely**. Diagnosed
against upstream `docker-compose.example.yml` — the official compose
mounts
[`docker/localstack_bootstrap.sh`](https://raw.githubusercontent.com/standardnotes/server/main/docker/localstack_bootstrap.sh)
into `/etc/localstack/init/ready.d/`, which creates the SNS topics
and SQS queues `standardnotes/server` workers expect
(`auth-local-queue`, `syncing-server-local-queue`,
`files-local-queue`, `revisions-server-local-queue`,
`analytics-local-queue`, `scheduler-local-queue`, plus matching
`*-local-topic` topics). Without this, LocalStack is reachable but
**empty**, and `standardnotes/server` workers loop on missing-queue
errors with no `ENOTFOUND` line — the exact reported symptom.

Changes in this pass:

- **`scripts/localstack_bootstrap.sh`** — new file. Copied verbatim
  from upstream `standardnotes/server` `docker/localstack_bootstrap.sh`
  (URL above). Header block notes the upstream source and AGPL-3.0
  licence inheritance. `chmod +x` on the file, `bash -n` validated.
- **`templates/standardnotes-localstack.xml`** — added a new required
  Path mapping *LocalStack Bootstrap Script* with
  `Target="/etc/localstack/init/ready.d/localstack_bootstrap.sh"` and
  `Default="/mnt/user/appdata/standardnotes/localstack_bootstrap.sh"`.
  Overview text gained a "REQUIRED INIT SCRIPT" block explaining what
  it does and giving the exact `curl` + `chmod` host-prep commands.
  XML re-validated with `ET.parse` after the edit.
- **`README.md`** —
  - § 0 *Common causes*: added "LocalStack reachable but uninitialised
    (no topics / queues)" bullet with the `awslocal sqs list-queues`
    / `sns list-topics` verification snippet.
  - § 0 *Emergency checklist*, step 5: now also verifies queues/topics
    exist, not just TCP reachability.
  - § 0 *Emergency bootstrap (LocalStack already running, no queues)*:
    new subsection with `docker cp` + `docker exec` recipe for an
    in-place fix on a running container.
  - § 3 Step 2: prepended the host-prep `mkdir` / `curl` / `chmod +x`
    commands and the post-start `awslocal` verification block.
- **`docs/configuration.md`** — new subsection *Required bootstrap
  (SNS topics / SQS queues)* under LocalStack, mirroring the Step 2
  install instructions and linking back to the emergency bootstrap.
- **`docs/sync-loop-troubleshooting.md`** —
  - § 2a: TCP/DNS checks now followed by a "TCP-reachable is not
    enough" verification of `awslocal sqs list-queues` / `sns
    list-topics`.
  - § 2a-bis (new): full troubleshooting checklist for the specific
    failure mode "TCP fine, hostname resolves, queues missing", with
    both the host-side fix and the in-place `docker cp` / `docker
    exec bash` emergency procedure.
  - § 6 log-table: added a row distinguishing missing-queue errors
    (no `ENOTFOUND`) from the § 2a `ENOTFOUND localstack` case.
- Preserved all prior guidance: static IP / `--add-host` for
  `br0`/macvlan/VLAN, `COOKIE_DOMAIN` shape, files port `3125:3104`,
  MariaDB-only DB, container names `StandardNotesServer` /
  `StandardNotes-LocalStack`.

Validation summary for this pass:

- `bash -n scripts/localstack_bootstrap.sh` → OK.
- `python3 xml.etree.ElementTree.parse` on both templates and both
  `.github/assets/*.svg` → all OK.
- `yaml.safe_load` on both `.github/workflows/*.yml` → both OK.
- `grep -rnE "REPLACE_WITH_[A-Z_]+" templates README.md docs scripts`
  → no hits.

Files changed in this pass:

- `scripts/localstack_bootstrap.sh` (new, executable)
- `templates/standardnotes-localstack.xml`
- `README.md`
- `docs/configuration.md`
- `docs/sync-loop-troubleshooting.md`
- `HANDOFF.md` (this section)

## Not done (out of scope per instructions)

- No `git remote add` / `git push`.
- No `gh repo create`.
- No registration with the Unraid Community Applications feed.
- No Unraid forum thread created (cannot be done from here).
