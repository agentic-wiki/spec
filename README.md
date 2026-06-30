# Agentic Wiki

A portable, agent-operated knowledge base: plain Markdown in a folder that a human, an LLM, and the `wiki` CLI all read the same way. One store holds your notes, documents, datasets, and backlog, and an agent can run the whole thing while you keep every file.

Karpathy's LLM-wiki note named the idea, a knowledge base an agent could actually live in. The idea resonated but a full version never shipped. This is the rest of the stack that makes it actionable.

## The stack: three layers

1. **Markdown (the format).** Plain files in a folder, with a light convention (a `type` in frontmatter, links between entries, two reserved filenames) compatible with the [Open Knowledge Format](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing). Complete and navigable on its own.
2. **`wiki` (the tool).** A neutral, deterministic engine that indexes the folder and answers structured queries, reports the link graph, and safely refactors. Built for agents to call, runnable by anyone.
3. **Skills.** The operating manual an agent follows to drive the tool, where workflow opinions live, and yours to customize.

The foundation is just your Markdown; agentic-wiki provides the top two layers, the tool and the skills.

**Separation principle:** the format stands alone, the tool is a neutral engine over it, and the skill is where opinion lives. Never let one layer do another's job.

## Why one plain-markdown store

Markdown in git is the only store that is human-readable, LLM-native, greppable, diffable, versioned, portable, self-hostable, and standard, all at once. Your IDE, Obsidian, the CLI, and any LLM open the very same files, and nothing is locked inside a proprietary app.

A plain folder, though, is dumb: `grep` cannot tell you what links to a page, which notes have no home, or move a file and fix the links behind it. `wiki` CLI adds exactly that structure, so you get the freedom of plain text and the power of a database at once.

## The format

### The mental framework

Three orthogonal axes classify every entry. Keeping them separate is what keeps the system friction-free.

| Axis | Question it answers | Cardinality | Home |
|---|---|---|---|
| **Folder path** | Where does this *one* file live? | exactly one | directory tree |
| **`type`** | What *kind* of entry is it? | exactly one | frontmatter |
| **Tags** | What *other dimensions* does it touch? | many | frontmatter |

**Domain rule: if you'd ever want a thing to appear in two places at once, it is a tag, not a folder.**

- **Prefer domain over type for folders.** A trip plan (concept), its flights (dataset), and the trip itself (event) all sit together in `personal/travel/`, sliced by `type` on demand, rather than scattered across `concepts/ events/ datasets/`. This is a lean, not a law (a `tasks/` folder that groups *actions* by actionability is a reasonable workflow choice).
- **A folder is a single, stable home.** The path is the entry's identity, so moving a file rewrites the links to it. Keep the top levels stable; `wiki move` rewrites links safely when you do move things.
- **Tags carry everything cross-cutting**: `database`, `2026`, `needs-review`, `feature`.

Most of these are judgment calls (whether something is a folder or a tag, which domain it belongs to) that no tool can make; you and the skill define them. `wiki check` verifies only the mechanical rules: every file carries a known `type`, links resolve, folders stay shallow.

### Bundle layout

A knowledge base is a **bundle**: a directory marked by a `wiki.toml` file, with the markdown content sitting directly inside it. It works like a git repo, the marker plus your files at the root, not a `wiki/` subfolder wrapping the content. Root-absolute links resolve from the bundle root, so `/finance/income.md` is the file `finance/income.md` beside `wiki.toml`.

```
my-kb/                       # the bundle root == the content root
├── wiki.toml                # marker + config (tracked)
├── .wiki/                   # query cache/index (gitignored, rebuilt on demand)
├── .gitignore
├── index.md                 (home page, links into each domain)
├── finance/
│   ├── index.md
│   ├── budget.md            (type: note)
│   └── expenses.md          (type: dataset)
├── personal/travel/
│   ├── index.md
│   ├── flights.md           (type: dataset)
│   └── copenhagen-2026.md   (type: event)
├── tech/infra/
│   ├── index.md
│   ├── reverse-proxy.md     (type: concept)
│   └── hetzner.md           (type: tool)
├── sources/
│   └── okf-spec.md          (type: source)
└── tasks/
    ├── index.md             (the board)
    └── renew-passport.md    (type: task)
```

- **`wiki.toml`** (tracked) marks the bundle root and declares config. The CLI walks up from the working directory to find it, the same way git finds `.git`. It is deliberately tiny:
  ```toml
  spec = "0.1"
  types = ["note", "concept", "entity", "event", "tool", "project", "task", "dataset", "source", "draft"]
  ```
- **`.wiki/`** is a disposable per-bundle cache, the same idea as `.git`: local, never committed, rebuildable.
- **Content** is every `.md` file under the bundle root. The scanner indexes only markdown and skips hidden directories, so the cache (`.wiki/`) and any viewer artifacts (`.obsidian/`) never become entries. Hand someone the bundle and they hold a clean, self-contained knowledge base; Obsidian opens it as a vault (configure markdown links and absolute-path-in-vault).

### Entries and frontmatter

Every markdown file is an entry with YAML frontmatter. The only required field is `type`, on every file.

```markdown
---
type: concept
title: Docker Networking
tags: [infra, docker, networking]
timestamp: 2026-06-24
---

A two-to-three sentence summary.

## Details
…

## See also
- [Container orchestration](/tech/infra/container-orchestration.md)
```

Standard optional fields (OKF): `title`, `description`, `tags`, `timestamp` (ISO 8601, validated when present). This profile adds `resource`: a pointer to the specific live thing a page represents (a server console, a dataset's table), distinct from a `source`'s `url`, which is provenance.

### Type vocabulary

Declared in `wiki.toml` and extensible: there are no reserved *types*. `index.md`/`log.md` are reserved *filenames* with no frontmatter (see Reserved files).

| type | For | Notable fields |
|---|---|---|
| `note` | a processed thought or observation | `status` |
| `concept` | an idea, technology, methodology | |
| `entity` | a person, org, product, place, named thing | |
| `event` | something that happened at a point in time | `date` |
| `tool` | software, hardware, utility | |
| `project` | something being built or planned | `status`, `priority`, `effort` |
| `task` | an action item | `status`, `priority`, `effort`, `due` |
| `dataset` | time-series / tabular source data | |
| `source` | an external reference (provenance) | `url`, `source_type` |
| `draft` | a transitional entry, not yet classified | |

The list is a starting vocabulary, not a mandate: extend or replace it in `wiki.toml`. `draft` is the inbox-capture convention (see the skills).

### Links

Standard markdown links can be **root-absolute** from the bundle root, `[Income](/finance/income.md)` (canonical, and preferred for readability and stability), or **relative**, `[a](./other.md)` (resolved against the file's directory). No wikilinks.

If you use Obsidian, set it to write standard markdown links: Settings → Files and links → turn off **Use [[Wikilinks]]**, set **New link format** to *Absolute path in vault*, turn on **Automatically update internal links**. Otherwise it emits `[[wikilinks]]`, which neither the format nor `wiki` recognizes, so those links silently vanish from the graph.

`wiki tidy --links` rewrites relative links to root-absolute. A broken link (target not in the bundle) is not malformed, it may be not-yet-written knowledge, so consumers tolerate it; `wiki check` and `wiki unresolved` surface them as a courtesy.

`resource:` is a frontmatter pointer, never a body link, and is not treated as an internal link. Entry filenames are slugs (lowercase, hyphenated, no spaces) so links stay simple bare paths; `wiki check` warns on a space in a path. (`wiki` does resolve `<…>`-wrapped links, markdown's way to allow a space, but `wiki check` still flags the spaced filename; `wiki tidy --slug` renames it to a slug, and naming files as slugs from the start avoids it.)

### Reserved files

They are **not concept documents**: no frontmatter, exempt from the `type` requirement, and skipped by `wiki orphans` (they are entry points or side-narratives).

- **`index.md`**: progressive-disclosure navigation for its folder (a list of links). Agents and humans read indexes top-down and follow links rather than grepping blindly.
- **`log.md`** _(optional)_: an append-only chronological changelog or narrative (what happened in this area over time), with ISO `YYYY-MM-DD` date headings. This is *not* data. Create it only when a domain benefits from an audit trail; it is not mandatory. `wiki check` and `wiki orphans` skip it.

### Datasets, logs, and tasks

Three different things, often confused:

- **Dataset** (`type: dataset`): tabular or time-series source data (flights, expenses, activity). **One dataset per file equals one table** (the header row names the fields); keep small data inline as a markdown table (`wiki table` extracts it as csv/json), and point `resource:` at an external CSV/sqlite for large data. Multiple datasets are multiple files, so extraction is never ambiguous.
- **`log.md`**: a changelog or narrative, not data (see Reserved files).
- **Tasks**: an inline `- [ ]` checkbox is the default, and most tasks stay inline. Promote one to an entry with `type: task` when it carries context an inline line cannot capture (a multi-step plan, a blocked or `due` status, links to other entries); not every entry needs subtasks. The work-kind (feature, bug, debt, chore) is a **tag**. The active list is a *board*: an `index.md` (or a `todo.md`) of checkboxes that link to the detailed entries.

  **Sync invariant**: when a task entry carries a `status` frontmatter field, the board's checkbox for that entry is checked exactly when `status` is `done`. Update both in the same change so the board never goes stale. `wiki tasks` surfaces open checkboxes; `wiki list --type task` lists all task entries regardless of checkbox state.

  **Pruning**: the board is a short-lived working surface; task entries are working records. When work ships, its lasting value is captured in `log.md` (a dated summary) or promoted to a permanent KB entry. The task file itself can then be deleted: it served its purpose. Your workflow may decide to use only checkboxes, only entries, or both; the tool is not prescriptive.

### Folders

Prefer a shallow **domain → subdomain** shape: 2-3 levels recommended, no hard cap (`wiki check` warns past three). The exact buckets are yours to choose. Coarse scoping is the folder's job; cross-cutting retrieval is the job of tags and links.

Folder names and entry filenames use **slugs**: lowercase, hyphenated words, no spaces, no underscores. `wiki check` warns on non-slug names; `wiki tidy --slug` rewrites them.

## The tool (`wiki`)

Why a tool at all: `grep` cannot compute a link graph (orphans, backlinks, unresolved) or answer structured frontmatter queries (by `type`/`tag`/field), and an agent would burn a lot of tokens reading content every time, of which only a fraction might be relevant. `wiki` is the deterministic engine in between: every query, move, and check is exact and repeatable, so an agent runs your base with the reliability of a database, not the guesswork of a prompt.

A single static Go binary, zero runtime dependencies, native on macOS, Linux, and Windows. It indexes a bundle and answers structured queries (`list`/`search` with `--type/--tag/--prefix`), introspects the vocabulary (`tags`, `properties`, `property`), reports the link graph (`links`, `backlinks`, `unresolved`, `orphans`), lists tasks, canonicalizes links and filenames (`tidy`), and safely refactors (`move` rewrites every link to a relocated entry). Output as text, JSON, CSV, or TSV.

**Exit codes:** `0` on success: enumeration and diagnostic commands (`list`, `tasks`, `orphans`, `unresolved`, …). `1`: `search` with no match (like `grep`), `table` with no such table, and `check` with conformance errors (like a linter). `2` on actual errors (missing bundle, bad arguments, unreadable file).

`wiki check` is an opt-in lint (broken links, missing or unknown types, depth, slug, `okf_version` drift, `timestamp` format), **not an OKF gate**: a bundle that does not pass `check` can still be valid OKF. `wiki init` scaffolds a conformant bundle from an embedded starter, so the tool is its own template generator and there is no template-versus-spec drift.

## The skill

The operating manuals an agent follows to drive `wiki`, defined once and surfaced per runtime, living in the [skills repo](https://github.com/agentic-wiki/skills). They encode the mental framework (the judgment the tool cannot make) and the data lifecycle. The format stays neutral; all workflow opinion lives here. Two ship today:

- **agentic-wiki**: operating a knowledge base. A set of independent operations the agent can enter at any point — **capture** a rough thought as a `type: draft` in `inbox/`; **refine** it by reading it back and sharpening; **promote** it (real `type`, `wiki move` into its domain, link it in); **retrieve** (search, list, follow links); **groom** (`check`, `tidy`, cross-link, groom orphans). A session might do just one of these, or chain several. Already fully formed? Skip the inbox and create directly.
- **agentic-backlog**: operating a task backlog that is itself a wiki bundle, with the entry `status` and its board checkbox kept in sync. Bring your own UI or agent on top.

The inbox can be a live queue of `type: draft` entries, so `wiki list --type draft` is the to-refine list; an item leaves the queue the moment it is promoted. A binary in flight (a PDF) can sit in a gitignored `inbox/resources/`, pointed at by the draft's `resource:`.

**The conventions are yours.** Beyond "every file has a `type`," the format fixes almost nothing; how you work the bundle is a skill-layer choice you customize. The inbox above is one example: have one or don't. Same for boards, naming, and ingestion flow.

## Scope

**In scope:** the markdown format, the `wiki` CLI, the agent skills, a minimal embedded scaffold.

**Out of scope:**

- **Binary document store.** The framework's job ends when knowledge is extracted into the wiki; where an original binary lives afterward (an external store, or deleted) is not its concern. The optional `resource:` field only *points* at it.
- **Sync.** Git handles it.
- **Encryption.** Staying git-based means git-crypt can drop in per folder. Defined in your agent skills.
- **History search.** Git is the time machine (`git log -S`, `git log --follow`, `git blame`); the tool indexes current state only.

## Status and ecosystem

Three single-purpose repos, each with its own release cadence:

- **`agentic-wiki/wiki`**: the CLI tool; embeds the starter so `wiki init` needs no network.
- **`agentic-wiki/template`**: the reference bundle `wiki init` emits.
- **`agentic-wiki/skills`**: the operating manuals, installed into an agent's skill directory, evolving independently of the tool.

The CLI is built, tested, and dogfooded. This repo dogfoods the format itself: its roadmap lives under [`tasks/`](tasks/index.md), an agentic-wiki bundle operated by `wiki` (`wiki tasks`, `wiki list --type task`, `wiki check`).

## Origins

Builds on Andrej Karpathy's [LLM-wiki idea](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f#file-llm-wiki-md) and Google's [Open Knowledge Format](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing).
