# Setup: automated notes

How Avram was set up (2026-08-29): a GitHub markdown catalogue, published with MkDocs, that Scratch Pad files into when Andrej or another agent sends findings.

## What it is

- Repo: [webduvet/avram](https://github.com/webduvet/avram)
- Site (after Pages is enabled): [webduvet.github.io/avram](https://webduvet.github.io/avram/)
- Agent: Scratch Pad (this bot). Other agents can send findings here; they get written as pages.

The left sidebar **is** the `docs/` folder. A directory is a topic, `index.md` is that topic’s landing page, any other `.md` file is a page. There is no separate nav config to keep in sync.

```
docs/note-taking/setup-automated-notes.md  →  Note taking / Setup: automated notes
docs/inbox/index.md                        →  Inbox
```

## Access (deliberately narrow)

Full GitHub account access was rejected. The bot uses a **fine-grained personal access token** on this repo only, stored in Grok Bot’s secret store (not in chat).

| Permission | Access | Why |
|---|---|---|
| Contents | read/write | commit pages |
| Pull requests | read/write | open PRs for bigger reorgs |
| Metadata | read (required) | GitHub default |
| Actions | read | see CI failures |
| Commit statuses | read | see checks |
| Deployments | read | see Pages/deploy status |

Not granted: Administration, Workflows write, Secrets, org-wide scopes. Classic PATs and GitHub OAuth were considered and skipped as too broad.

[Fine-grained personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens)

## How notes get filed

1. Andrej or another agent sends findings to Scratch Pad.
2. If the topic is clear, a page goes under `docs/<topic>/`. If not, it goes to [Inbox](../inbox/index.md).
3. Ordinary new notes commit to `main` (so Pages can publish). Large moves/renames go through a pull request.
4. Filename and folder are kebab-case. Links between notes are normal relative markdown, not wikilinks.

## Site stack

[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/). `mkdocs.yml` has **no `nav:`** key; Material’s `navigation.indexes` makes each folder’s `index.md` the section page.

Local preview:

```sh
pip install -r requirements.txt
mkdocs serve
```

Deploy: `.github/workflows/pages.yml` builds on push to `main` and deploys to GitHub Pages. One-time repo setting: **Settings → Pages → Source: GitHub Actions**. Scaffold: [PR #1](https://github.com/webduvet/avram/pull/1) (merged).

## What was considered

**Quartz** ([demo](https://quartz.jzhao.xyz/), [repo](https://github.com/jackyzha0/quartz)) — markdown digital garden: `[[wikilinks]]`, backlinks, graph. A better fit if notes were going to be densely cross-linked. They will not be, much. MkDocs already does ordinary relative links. Quartz’s extra chrome is unused cost.

**Starlight** (Astro) — polished documentation site. Heavier than a personal catalogue.

**MkDocs Material** — chosen. Already known, simple look, easy to navigate, and the file tree matches the web UI 1:1.

**Obsidian Publish / GitBook** — not in the running: paid or not git-first.

## Still to do

Enable GitHub Pages (Actions source) if the public site is not live yet.
