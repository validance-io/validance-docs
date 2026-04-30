# validance-docs

Source for [docs.validance.io](https://docs.validance.io) — the public Validance documentation site.

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/). Deployed via GitHub Actions on push to `main`.

## Local preview

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install mkdocs-material
mkdocs serve
```

Open http://127.0.0.1:8000.

## Project layout

```
docs/          # markdown source (one file per page)
mkdocs.yml     # site config (nav, theme, extensions)
.github/       # CI workflow that builds and deploys on push to main
```

## Adding a page

1. Create `docs/<slug>.md`
2. Add it to `nav:` in `mkdocs.yml`
3. Open a PR — the workflow will build with `strict: true` (broken internal links fail the build)
4. Merge to `main` — the site auto-deploys

## License

Documentation content © Validance. The site source (config, theme overrides) is MIT.
