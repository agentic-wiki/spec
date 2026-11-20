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

This is version **0.1** of the format: a bundle declares it with `spec = "0.1"` in its `wiki.toml`. Changes are listed in [CHANGELOG.md](CHANGELOG.md).

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
├── .wiki/                   # reserved for a local cache (gitignored)
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
    ├── index.md             (the task list)
    └── renew-passport.md    (type: task)
```

- **`wiki.toml`** (tracked) marks the bundle root. The CLI walks up from the working directory to find it, the same way git finds `.git`. It holds the bundle's config: see [`wiki.toml`](#wikitoml).
- **`.wiki/`** is reserved for a tool's local cache, the same idea as `.git`: never committed, and safe to delete.
- **Content** is every `.md` file under the bundle root. The scanner indexes only markdown, skips hidden directories, any folder holding its own `wiki.toml` (a nested bundle, below; `wiki` v0.19.0 and later), and any path matched by `wiki.toml`'s `ignore`, so the cache (`.wiki/`), viewer artifacts (`.obsidian/`), and declared non-entries (`AGENTS.md`, a `README`) never become entries — absent from `list`/`search`/the graph and raising no `check` issue, though a link *to* an ignored file still resolves on disk and so is not broken. Hand someone the bundle and they hold a clean, self-contained knowledge base; Obsidian opens it as a vault (set it to write standard links: see [Links](#links)).
- **Nested bundles** (`wiki` v0.19.0 and later). A folder holding its own `wiki.toml` is a separate bundle, with its own config. The bundle around it skips that folder entirely, the way git skips a nested repository: its files are not entries of the outer bundle. For links into it, see [Links](#links).

### `wiki.toml`

```toml
spec = "0.1"
types = ["note", "concept", "entity", "event", "tool", "project", "task", "dataset", "source", "draft"]
ignore = ["AGENTS.md", "CLAUDE.md", "WORKFLOW.md"]
ignore_orphans = ["tasks/**"]
```

The keys `wiki` reads:

| Key | Type | Meaning |
|---|---|---|
| `spec` | string | The spec version the bundle follows. Each version embeds an OKF version (`0.1` embeds OKF `0.1`), which the root `index.md` declares as `okf_version` (see Reserved files). |
| `types` | string[] | The allowed `type` values. Left out, any type is allowed. |
| `ignore` | string[] | Paths that are *not* content, relative to the bundle root: agent-operating files like `AGENTS.md`, a `README`, a `LICENSE`. Inside the bundle, a match is left out of the index (see Content): an exact path, a `dir/**` subtree, or any `*`, `?` or `**` pattern. An exact path outside it (`../PRD.md`) acknowledges an out-of-bundle link (see Links). |
| `ignore_orphans` | string[] | Entries nothing has to link to. They stay indexed, but `wiki orphans` does not report them. An exact path, a `dir/**` subtree, or any `*`, `?` or `**` pattern. |

`wiki check` warns on any other key, so a typo or a renamed key never fails silently. It also warns on an `ignore` or `ignore_orphans` pattern inside the bundle that matches no file (`wiki` v0.19.0 and later): `ignore = ["drafts"]` names a file, not the folder, so write `drafts/**`. Two kinds of keys are set aside for apps instead: `wiki` never reads inside them and never warns about them.

**`[tool.<name>]`** is one tool's own config, as in `pyproject.toml`. The tool picks its name and its keys, and validates them itself. One file then describes the bundle, instead of a config file per tool.

```toml
[tool.wikanban]
group_by = "status"
```

**`[[board]]` and `[[graph]]`** are views that any app may show. Each table names a view over one folder.

```toml
[[board]]
id      = "board"
path    = "/tasks"
where   = ["type=task"]
columns = ["todo", "in-progress", "done"]
lane    = "priority"

[[graph]]
id         = "people"
path       = "/people"
where      = ["type=entity"]
neighbours = true
```

A `[[board]]` shows the entries under a folder as cards in columns:

| Key | Type | Required | Meaning |
|---|---|---|---|
| `id` | string | yes | The board's name in addresses: lowercase letters, digits, `_` and `-`, starting with a letter or a digit (`^[a-z0-9][a-z0-9_-]*$`). Unique among boards. |
| `path` | string | yes | The folder, from the bundle root (`/tasks`; `/` is the whole bundle). Cards are the entries anywhere under it. |
| `name` | string | no | The title on screen. Left out, the app names it from the folder. |
| `where` | string[] | no | Filters, in `wiki list --where` spelling, all must match. Left out: `["type=task"]`. `[]`: every entry under `path`. |
| `status` | string | no | The frontmatter field the columns come from. Default `status`. |
| `columns` | string[] | no | Sets the order and adds columns that may be empty. It never hides a value: one found but not listed still gets a column. |
| `lane` | string | no | The field that groups cards into rows. Left out: no lanes. |
| `lanes` | string[] | no | Lane order, by the rule of `columns`. |
| `blockers` | string | no | The field naming what a card waits on. Default `blockers`. |

A `[[graph]]` shows the entries under a folder and the links between them:

| Key | Type | Required | Meaning |
|---|---|---|---|
| `id` | string | yes | As a board's. Graph ids are their own namespace: a graph may share an id with a board. |
| `path` | string | yes | As a board's. |
| `name` | string | no | As a board's. |
| `where` | string[] | no | As a board's, but with no default: left out means every entry under `path`. |
| `neighbours` | bool | no | Also show the entries one link away. Default `false`. |

An app that reads them:

- may ignore a table or a key it does not show, but must not report a key of these tables as unknown;
- reports what is wrong (a missing `id`, a filter that does not parse, a repeated id, an unknown key) and still shows what it can: it never refuses to open the bundle over them.

### Entries and frontmatter

Every markdown file is an entry with YAML frontmatter. The only required field is `type`, on every file but the reserved ones (see Reserved files).

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

**Header layout.** Values can be any YAML: text, numbers, `true`/`false`, null, lists, nested maps, multi-line strings. The layout of the header is fixed, so any tool can change one key without rewriting the rest:

- each top-level key is plain text on its own line, at column 0;
- no flow-style header (`{type: note, title: X}`);
- no `<<` merge keys;
- each key written once, and the header at most 1 MiB.

`wiki check` reports a header that breaks these rules as an error.

### Type vocabulary

Declared in `wiki.toml` and extensible: there are no reserved *types*. `index.md`/`log.md` are reserved *filenames* with no `type` (see Reserved files).

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

Entries link with standard markdown links. The canonical form is **relative** to the linking file, `[Income](../finance/income.md)`, so links work in any markdown viewer, GitHub included. A **root-absolute** link, `[Income](/finance/income.md)`, resolves from the bundle root and is valid too; `wiki tidy --links` rewrites it relative, and `wiki move` writes relative links.

Every reader resolves a link's destination the same way:

- its path is percent-decoded once (`my%20file.md` names `my file.md`), and a `?query` is dropped;
- `<…>` around it is markdown's way to allow a space, and is read the same way;
- a destination starting with a scheme (`https:`) or `//` is external, never an entry;
- a `#anchor` names a heading of the target by its slug: lowercased, each space a `-`, letters, digits, `-` and `_` kept, anything else dropped (`## Docker Networking` is `#docker-networking`). `wiki check` warns on an anchor no heading has. An Obsidian block reference (`#^id`) is left alone.

A broken link (target not in the bundle) is not malformed, it may be not-yet-written knowledge, so consumers tolerate it; `wiki check` and `wiki unresolved` surface them as a courtesy. A link that resolves *outside* the bundle (climbing above the root, e.g. `../PRD.md` from a nested bundle) is neither an edge nor broken; `wiki check` notes it as an out-of-bundle advisory, and listing that path in `wiki.toml`'s `ignore` silences it. Such a path is never resolved on disk (the containment guard that also blocks path traversal).

A link to a file inside the bundle that is not an entry, because `ignore` matches it or because it sits in a nested bundle (`wiki` v0.19.0 and later), is read like a link to an image: it resolves when the file exists on disk, so it is not broken and `wiki check` says nothing about it, but it is not an edge of the graph and its `#anchor` is not checked. When the file does not exist, it is a broken link. There is nothing to silence.

**Frontmatter references.** A frontmatter value whose path ends in `.md` (`epic: /backlog/epics/search.md`, `blockers: [/backlog/login.md]`) is a reference to that entry. It is written **root-absolute**, because it is a key to match on, not a link to click: every entry that points at one epic spells it the same way, so `wiki list --where epic=/backlog/epics/search.md` finds them all. A body link is relative because it must navigate; a frontmatter reference is root-absolute because it must be a stable key. `wiki move --include-frontmatter` rewrites them too. `resource:` points at something outside the bundle and is never a link.

**Wikilinks** (`[[page]]`) are not part of the format. `wiki` reads them for compatibility, the way Obsidian resolves them (with `aliases:` frontmatter), warns about them in `wiki check`, and converts them to standard links with `wiki tidy --wikilinks`. If you use Obsidian, set it to write standard links: Settings → Files and links → turn off **Use [[Wikilinks]]**, set **New link format** to *Relative path to file*, and turn on **Automatically update internal links**.

Entry filenames are best written as slugs (see Folders), so links stay plain paths.

### Reserved files

They are **not concept documents**: no `type` and no other frontmatter, and `wiki orphans` never reports them (they are entry points or side-narratives). One exception: the bundle-root `index.md` carries `okf_version`, the OKF version its `spec` embeds (see `wiki.toml`). `wiki init` writes it, `wiki check` warns when it is missing or out of sync, and `wiki check --fix` writes it.

- **`index.md`**: progressive-disclosure navigation for its folder (a list of links). Agents and humans read indexes top-down and follow links rather than grepping blindly.
- **`log.md`** _(optional)_: an append-only chronological changelog or narrative (what happened in this area over time), with ISO `YYYY-MM-DD` date headings. This is *not* data. Create it only when a domain benefits from an audit trail; it is not mandatory. `wiki check` warns on a date heading that is not ISO.

### Datasets, logs, and tasks

Three different things, often confused:

- **Dataset** (`type: dataset`): tabular or time-series source data (flights, expenses, activity). **One dataset per file equals one table** (the header row names the fields); keep small data inline as a markdown table (`wiki table` extracts it as csv/json), and point `resource:` at an external CSV/sqlite for large data. Multiple datasets are multiple files, so extraction is never ambiguous.
- **`log.md`**: a changelog or narrative, not data (see Reserved files).
- **Tasks**: an inline `- [ ]` checkbox is the default, and most tasks stay inline. Promote one to an entry with `type: task` when it carries context an inline line cannot capture (a multi-step plan, a blocked or `due` status, links to other entries); not every entry needs subtasks. The work-kind (feature, bug, debt, chore) is a **tag**. The active list is a *task list*: an `index.md` (or a `todo.md`) of checkboxes that link to the detailed entries. (A *board* is a view of entries by status, declared with `[[board]]` in `wiki.toml`.)

  **Sync invariant**: when a task entry carries a `status` frontmatter field, the task list's checkbox for that entry is checked exactly when `status` is `done`. Update both in the same change so the list never goes stale. `wiki checkboxes` surfaces open checkboxes; `wiki list --where type=task` lists all task entries regardless of checkbox state.

  **Pruning**: the task list is a short-lived working surface; task entries are working records. When work ships, its lasting value is captured in `log.md` (a dated summary) or promoted to a permanent KB entry. The task file itself can then be deleted: it served its purpose. Your workflow may decide to use only checkboxes, only entries, or both; the tool is not prescriptive.

### Folders

Prefer a shallow **domain → subdomain** shape: 2-3 levels recommended, no hard cap (`wiki check` warns past three). The exact buckets are yours to choose. Coarse scoping is the folder's job; cross-cutting retrieval is the job of tags and links.

Folder names and entry filenames are best written as **slugs**: lowercase, hyphenated words. That is a recommendation, not a rule. The rule is **no spaces**, because a space breaks a plain link: `wiki check` warns on a space anywhere in a path, and `wiki tidy --slug` replaces it with hyphens: in a file name, and in a folder name too (`wiki` v0.19.0 and later).

### Conformance

A bundle conforms when `wiki check` reports no error. The errors are:

- a file that cannot be read;
- frontmatter that is not valid YAML, writes a key twice, or exceeds the size limits;
- a header that breaks the header layout (see Entries and frontmatter);
- an entry with no `type`, or with a `type` that `wiki.toml` does not declare;
- a `timestamp` that is not ISO 8601.

Everything else `wiki check` reports is a warning, advice that never fails the check. The warnings cover unknown `wiki.toml` keys and `ignore` patterns that match nothing (`wiki` v0.19.0 and later), frontmatter on a reserved file, log headings that aren't ISO dates, folders deeper than three, spaces in paths, wikilinks, broken links and anchors, out-of-bundle links, and a missing or stale `okf_version`.

`wiki check` is a lint, **not an OKF gate**: a bundle that does not pass it can still be valid OKF.

## The tool (`wiki`)

Why a tool at all: `grep` cannot compute a link graph (orphans, backlinks, unresolved) or answer structured frontmatter queries (by `type`/`tag`/field), and an agent would burn a lot of tokens reading content every time, of which only a fraction might be relevant. `wiki` is the deterministic engine in between: every query, move, and check is exact and repeatable, so an agent runs your base with the reliability of a database, not the guesswork of a prompt.

A single static Go binary, zero runtime dependencies, native on macOS, Linux, and Windows. It queries entries (`list`, `search`, filtered with `--where key=value` and `--prefix`), reads them (`read`, `outline`, `table`, `dump`), introspects the vocabulary (`tags`, `properties`, `property`), reports the link graph (`links`, `backlinks`, `unresolved`, `orphans`), lists checkboxes (`checkboxes`), canonicalizes links and filenames (`tidy`), safely refactors (`move` rewrites every link to a relocated entry), and lints (`check`). `wiki init` writes a new conformant bundle from a starter embedded in the tool. Output as text, JSON, JSONL, CSV, or TSV.

Every command, flag, output column and exit code is documented in the tool's README: [Everything it does](https://github.com/agentic-wiki/wiki#everything-it-does).

## The skill

The operating manuals an agent follows to drive `wiki`, defined once and surfaced per runtime, living in the [skills repo](https://github.com/agentic-wiki/skills). They encode the mental framework (the judgment the tool cannot make) and the data lifecycle. The format stays neutral; all workflow opinion lives here. Two ship today:

- **agentic-wiki**: operating a knowledge base. A set of independent operations the agent can enter at any point — **capture** a rough thought as a `type: draft` in `inbox/`; **refine** it by reading it back and sharpening; **promote** it (real `type`, `wiki move` into its domain, link it in); **retrieve** (search, list, follow links); **groom** (`check`, `tidy`, cross-link, groom orphans). A session might do just one of these, or chain several. Already fully formed? Skip the inbox and create directly.
- **agentic-backlog**: operating a task backlog that is itself a wiki bundle, with the entry `status` and its task-list checkbox kept in sync. Bring your own UI or agent on top.

The inbox can be a live queue of `type: draft` entries, so `wiki list --where type=draft` is the to-refine list; an item leaves the queue the moment it is promoted. A binary in flight (a PDF) can sit in a gitignored `inbox/resources/`, pointed at by the draft's `resource:`.

**The conventions are yours.** Beyond "every file has a `type`," the format fixes almost nothing; how you work the bundle is a skill-layer choice you customize. The inbox above is one example: have one or don't. Same for task lists, naming, and ingestion flow.

## Scope

**In scope:** the markdown format, the `wiki` CLI, the agent skills, a minimal embedded scaffold.

**Out of scope:**

- **Binary document store.** The framework's job ends when knowledge is extracted into the wiki; where an original binary lives afterward (an external store, or deleted) is not its concern. The optional `resource:` field only *points* at it.
- **Sync.** Git handles it.
- **Encryption.** Staying git-based means git-crypt can drop in per folder. Defined in your agent skills.
- **History search.** Git is the time machine (`git log -S`, `git log --follow`, `git blame`); the tool indexes current state only.

## Status and ecosystem

Four repos, each with its own release cadence:

- **`agentic-wiki/spec`**: this format definition.
- **`agentic-wiki/wiki`**: the CLI tool; embeds the starters, so `wiki init` needs no network.
- **`agentic-wiki/skills`**: the operating manuals, installed into an agent's skill directory, evolving independently of the tool.
- **`agentic-wiki/wiki-shell`**: WikiShell, a web UI for a bundle: read it in a browser, follow the links, tick the checkboxes, and see its boards and graphs.

The CLI is built, tested, and dogfooded. This repo dogfoods the format itself: its roadmap lives under [`tasks/`](tasks/index.md), an agentic-wiki bundle operated by `wiki` (`wiki checkboxes`, `wiki list --where type=task`, `wiki check`).

## Origins

Builds on Andrej Karpathy's [LLM-wiki idea](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f#file-llm-wiki-md) and Google's [Open Knowledge Format](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing).
