<p align="center"><img src="https://raw.githubusercontent.com/go-proc/brand/main/social/go-proc.png" alt="go-proc/docs" width="720"></p>

# go-proc/docs

Versioned documentation for [go-proc](https://github.com/go-proc), built with
[MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and versioned with
[mike](https://github.com/jimporter/mike). Published to the `gh-pages` branch and
served at <https://go-proc.github.io/docs/>.

The organization landing page ([go-proc.github.io](https://go-proc.github.io))
links here.

## Local preview

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
mkdocs serve                       # http://localhost:8000 (current sources)
mike serve                         # preview the versioned site
```

## Releasing a new docs version

```bash
mike deploy --push --update-aliases <version> latest
mike set-default --push latest
```
