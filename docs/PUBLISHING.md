# Publishing Checklist — Standard Notes for Unraid

Internal notes for taking this template from "local working directory"
to "listed in the Unraid Community Applications feed". Practical, in
order, German/English friendly.

---

## 1. Repo State

Prerequisites before anything else can happen:

- Repository is **public** on GitHub at
  `https://github.com/junkerderprovinz/standardnotes`.
- `main` branch builds cleanly — no uncommitted placeholder strings
  (`REPLACE_WITH_*`) anywhere in `templates/`, `README.md`, or `docs/`.
- The `validate` GitHub Actions workflow (`.github/workflows/validate.yml`)
  is green on `main`.
- `LICENSE` is in place (MIT for the wrapper, with the bundled-software
  notice intact).

Quick check from a clean clone:

```bash
git clone https://github.com/junkerderprovinz/standardnotes.git
cd standardnotes
grep -rn "REPLACE_WITH_" .   # must return nothing
python3 -c "import xml.etree.ElementTree as ET; \
  [ET.parse(f) for f in ['templates/standardnotes-server.xml', \
                          'templates/standardnotes-localstack.xml', \
                          '.github/assets/banner.svg', \
                          '.github/assets/icon.svg']]"
```

### Install command (templates-user destination)

After the repo is public, the canonical install commands users will
run on the Unraid console are:

```bash
mkdir -p /boot/config/plugins/dockerMan/templates-user

curl -fsSL -o /boot/config/plugins/dockerMan/templates-user/my-StandardNotes-Server.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes/main/templates/standardnotes-server.xml

curl -fsSL -o /boot/config/plugins/dockerMan/templates-user/my-StandardNotes-LocalStack.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes/main/templates/standardnotes-localstack.xml
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
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes/main/templates/standardnotes-server.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes/main/templates/standardnotes-localstack.xml \
  https://raw.githubusercontent.com/junkerderprovinz/standardnotes/main/.github/assets/icon.svg
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

The CA feed lives at
<https://github.com/Squidly271/AppFeed>. To list this template:

1. Fork `Squidly271/AppFeed`.
2. Add an entry pointing to the raw `standardnotes-server.xml` (and
   optionally `standardnotes-localstack.xml`) under the appropriate
   category (`Productivity` / `Cloud`).
3. Open a PR against upstream. Include:
   - Link to the GitHub repo.
   - Link to the forum support thread.
   - Confirmation that the template parses and pulls a real Docker
     image.
4. Address review feedback. Squid is responsive but strict about
   correctness — expect at least one round.

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
