# Avram

Catalogue of notes, published with [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

The website sidebar is the `docs/` folder tree. A directory is a topic, `index.md` is that topic’s landing page, and any other `.md` file is a page.

```
docs/
  index.md                 home
  inbox/index.md           incoming notes
  example-topic/
    index.md               topic landing page
    sample-chapter.md      a page under that topic
```

## Local preview

```sh
pip install -r requirements.txt
mkdocs serve
```

Then open http://127.0.0.1:8000/.

## Publish

On merge to `main`, GitHub Actions builds the site. In the repo: **Settings → Pages → Source: GitHub Actions**. The site is https://webduvet.github.io/avram/.
