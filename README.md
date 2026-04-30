# validance-docs

Source for [docs.validance.io](https://docs.validance.io) — the public Validance documentation site.

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/). Deployed via GitHub Actions on push to `main`.

## Local preview

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Open <http://127.0.0.1:8080>. The port (8080) is set in `mkdocs.yml` (`dev_addr`) to avoid colliding with Validance's reserved ports (8000/8001) on the dev VM.

## Project layout

```
docs/                 markdown source
  index.md            site landing
  concepts/           workflows, tasks, runs, audit-and-evidence
  compliance/         regulatory-mapping
mkdocs.yml            site config (nav, theme, extensions)
requirements.txt      build dependencies
.github/workflows/    CI workflow that builds and deploys on push to main
```

## Adding a page

1. Create `docs/<section>/<slug>.md`
2. Add it to `nav:` in `mkdocs.yml`
3. Open a PR — the workflow builds with `strict: true` (broken internal links fail the build)
4. Merge to `main` — the site auto-deploys
