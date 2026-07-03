# DataX Design

This repository hosts design artifacts for the DataX application, including
architecture documents, flow charts, sequence diagrams, container diagrams, and
architecture decision records.

The documentation site is built with MkDocs Material and published through
GitHub Pages.

## Local Development

Install dependencies and serve the site:

```bash
uv sync
uv run mkdocs serve
```

Build the site in strict mode:

```bash
uv run mkdocs build --strict
```

## Published Site

GitHub Pages is configured to publish from GitHub Actions:

https://mbellary.github.io/datax-design/
