# Publishing Checklist — Standard Notes for Unraid

Internal notes for taking this template from "local working directory"
to "listed in the Unraid Community Applications feed". Practical, in
order, German/English friendly.

---

## 0. Template Scope — Which Repo Ships Which Template

To avoid double-listing and stay aligned with Unraid CA expectations,
the two GitHub repos own a fixed, non-overlapping set of templates:

| Repo | CA templates published |
|---|---|
| [`junkerderprovinz/standardnotes-server`](https://github.com/junkerderprovinz/standardnotes-server) (this repo) | **`StandardNotesServer`** — official `standardnotes/server` backend.<br>**`StandardNotes-LocalStack`** — Standard-Notes-specific LocalStack companion (SNS/SQS) required by the official server image. |
| [`junkerderprovinz/standardnotes-webui`](https://github.com/junkerderprovinz/standardnotes-webui) | **`StandardNotes`** — official `standardnotes/web` browser client. |

### Why LocalStack lives in *this* repo, not a separate LocalStack repo

Generic Unraid templates for `localstack/localstack` may exist (or get
published) elsewhere. This repo deliberately ships its **own**
LocalStack template — `StandardNotes-LocalStack` — because it is
Standard-Notes-specific:

- It mounts the upstream
  [`localstack_bootstrap.sh`](../scripts/localstack_bootstrap.sh)
  init script into `/etc/localstack/init/ready.d/` to pre-create the
  exact set of SNS topics and SQS queues the official
  `standardnotes/server` workers expect (`auth-local-queue`,
  `syncing-server-local-queue`, `files-local-queue`,
  `revisions-server-local-queue`, `analytics-local-queue`,
  `scheduler-local-queue`, plus matching topics).
- It documents the **required `localstack` hostname mapping** for
  `br0` / macvlan / VLAN / static-IP Unraid setups
  (`--add-host=localstack:<LocalStack-IP>` on the server container),
  which a generic LocalStack template would have no reason to ship.
- It is versioned and updated together with the server template so
  the two stay in lockstep — bumping the server image will not break
  the companion's bootstrap expectations.

Pairing them in one repo (and one CA submission) keeps the install
path "one Apps page, two clicks" and avoids cross-repo drift.

### How users see this in the Unraid Apps tab

CA indexes templates **per template file**, not per repo. A single
GitHub repo containing multiple `templates/*.xml` files is published
as **multiple separate installable apps** on the Apps tab, each with
its own *Install* button, even though all of them share one
`ca_profile.xml` maintainer entry. For this repo that means:

- `StandardNotesServer` — appears as its own app card.
- `StandardNotes-LocalStack` — appears as its own app card.

Both link back to the same `ca_profile.xml` Profile / WebPage / Repo
(this repo). Users install them **one at a time**, in the order
documented in the README's *Install order (mandatory)* section
(MariaDB → Redis → LocalStack → Server → WebUI).

The browser client (`StandardNotes`) is **not** part of this CA
submission — it ships from the separate `standardnotes-webui` repo
and is published as its own CA submission. CA users will therefore
see three Standard-Notes-related app cards in total: two from this
repo, one from the WebUI repo.

*(The exact UI grouping CA chooses — e.g. whether the two templates
in this repo are visually grouped under the maintainer or listed
alongside other Productivity apps — can vary across CA UI versions.
What matters is that each template installs as its own container.)*

---

## 1. Repo State

Prerequisites before anything else can happen:

- Repository is **public** on GitHub at
  `https://github.com/junkerderprovinz/standardnotes-server`.
- `main` branch builds cleanly — no uncommitted placeholder strings
  (`REPLACE_WITH_*`) anywhere in `templates/`, `README.md`, or `docs/`.
- The `validate` GitHub Actions workflow (`.github/workflows/validate.yml`)
  is green on `main`.
- `LICENSE` is in place (MIT for the wrapper, with the bundled-software
  notice intact).

Quick check from a clean clone:

```bash
git clone https://github.com/junkerderprovinz/standardnotes-server.git
cd standardnotes
grep -rn "REPLACE_WITH_" .   # must return nothing
python3 -c "import xml.etree.ElementTree as ET; \
  [ET.parse(f) for f in ['templates/standardnotes-server.xml', \
                          'templates/standardnotes-localstack.xml', \
                          'ca_profile.xml', \
                          '.github/assets/banner.svg', \
                          '.github/assets/icon.svg']]"
```

### Install command (templates-user destination)

After the repo is public, the canonical install commands users will
run on the Unraid console are:

```bash
mkdir -p /boot/config/plugins/dockerMan/templates-user

curl -fsSL -o /boot/config/plugins/dockerMan/templates-user/my-StandardNotes-Server.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes-server/main/templates/standardnotes-server.xml

curl -fsSL -o /boot/config/plugins/dockerMan/templates-user/my-StandardNotes-LocalStack.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes-server/main/templates/standardnotes-localstack.xml
```

Unraid's **Docker → Add Container → Template → User templates**
dropdown picks them up automatically once the files exist in
`/boot/config/plugins/dockerMan/templates-user/`.

---

## 2. Verify Raw URLs

Both XML templates reference raw GitHub URLs for `<TemplateURL>` and
`<Icon>`. After the repo is public, fetch each one and confirm a `200`:

```bash
for url in \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes-server/main/templates/standardnotes-server.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes-server/main/templates/standardnotes-localstack.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes-server/main/.github/assets/icon.svg
do
  echo "$url"
  curl -fsI "$url" | head -n 1
done
```

A `404` here is the most common reason a CA submission is rejected —
fix before continuing.

---

## 3. Open an Unraid Forum Support Thread

CA expects each template to have a dedicated support thread on the
Unraid forums. Create one before submitting:

1. Go to <https://forums.unraid.net/forum/38-docker-engine/> and start a
   new topic.
2. Title suggestion: *Support — Standard Notes (Community Template)*.
3. Body: short description, link back to the GitHub repo, the README
   anchor for Quick Start, and a note that MariaDB and Redis are
   user-supplied.
4. Copy the resulting topic URL.

Then update both templates' `<Support>` tag from the GitHub Issues
fallback to the new forum URL. Keep Issues as the secondary channel in
the README:

```bash
# templates/standardnotes-server.xml
# templates/standardnotes-localstack.xml
#   <Support>https://forums.unraid.net/topic/<NUMBER>-<SLUG>/</Support>
```

Commit, push, and wait for the `validate` workflow to go green.

---

## 4. Submit to Community Applications

CA submissions go through the official submission form at
<https://ca.unraid.net/submit>. Before opening it, confirm:

- The repo is **public** and the `validate` workflow is green on `main`.
- An **OSI-approved license** (`LICENSE`) is at the repo root — this
  wrapper is MIT; upstream image licenses retain their own terms.
- Every template in `templates/` parses and pulls a real image, and
  every `<TemplateURL>` / `<Icon>` raw URL returns 200 (see § 2).
- A `ca_profile.xml` exists at the repo root with a **non-empty
  `<Profile>`** section, valid `<Icon>` / `<WebPage>` / `<Repo>` URLs,
  and the maintainer name visible on GitHub.
- An Unraid forum **support thread** exists and is linked from each
  template's `<Support>` tag (see § 3).

Then open <https://ca.unraid.net/submit> and submit **this repo** — it
publishes both `StandardNotesServer` and `StandardNotes-LocalStack` as
companion templates. The browser client lives in the companion repo
[`standardnotes-webui`](https://github.com/junkerderprovinz/standardnotes-webui)
and is submitted **separately** (one submission per repo, not one per
template).

CA review is responsive but strict about correctness — expect at
least one round of feedback. Address it on `main`, push, and reply
on the submission thread.

---

## 5. Tag a First Release

Once the CA PR is merged (or simultaneously, if you are confident):

```bash
git tag -a v0.1.0 -m "Initial Unraid Community Template release"
git push origin v0.1.0
```

Then on GitHub: **Releases → Draft a new release → v0.1.0**, paste a
short changelog (what's in the templates, what's intentionally not), and
publish. This gives users a stable reference point and surfaces the
project on the GitHub sidebar.

---

## Maintenance Notes

- **Watch upstream** `standardnotes/server` for breaking env-var
  renames; the template's `<Config>` keys must track them.
- **Re-run validation** after every template edit — the GitHub Actions
  workflow does this automatically on push / PR.
- **No bundled secrets, ever.** `examples/.env.example` is a template
  with placeholders only. If you ever paste a real secret while
  testing, scrub the working tree (`git reset`) before committing.
- **Banner / icon updates** should keep the existing neutral dark
  palette (`#232629`, `#3daee9`, `#bdc3c7`, `#fcfcfc`) for visual
  consistency in the GitHub README. This palette is repo-decorative
  only — Standard Notes is not themed around Breeze Dark, and the
  product itself has no KDE / Breeze relationship.
