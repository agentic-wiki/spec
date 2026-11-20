# Changelog

Changes to the format. A bundle declares the version it follows with `spec = "…"` in its `wiki.toml`.

## 0.1, 2026-11-20

The spec brought in line with `wiki` v0.18.0. It also states one rule that `wiki` follows from v0.19.0: `ignore` patterns that match nothing, and two from v0.20.0: dot files are not content, and neither is a link to a file outside the bundle or on a dot path. Two more come in a later release: nested bundles, and `tidy --slug` on folder names. Each is marked in the README. Still version 0.1: no bundle that conformed before stops conforming.

### Added

- **`[[board]]` and `[[graph]]`** in `wiki.toml`: views over a folder, defined by the standard for the apps that show them. `wiki` sets them aside and never reads inside them.
- **`[tool.<name>]`** in `wiki.toml`: one tool's own config, which `wiki` never reads.
- **The `wiki.toml` keys** in one table: `spec`, `types`, `ignore` and `ignore_orphans`. `ignore_orphans` was missing.
- **Header layout:** values can be any YAML, but each top-level key is plain text on its own line at column 0, with no flow-style header and no `<<` merge keys.
- **Conformance:** the errors that fail `wiki check`, apart from the warnings that never do.
- **How a link is read:** percent-decoding, `?query`, external destinations, and `#anchor` slugs.
- **Frontmatter references:** a value ending in `.md` names an entry and is written root-absolute.
- **`ignore` patterns that match no file** are warned about (`wiki` v0.19.0 and later).
- **The version** of the format, stated in the README, and this changelog.

### Changed

- **Dot files are not content**, as hidden folders already weren't: a path with any segment starting with `.` (`.notes.md`, `a/.hidden.md`) is never an entry (`wiki` v0.20.0 and later).
- **A link to a file that isn't content is not content**: a `.md` whose real file, symlinks resolved, is outside the bundle or on a dot path (a link to `~/.bashrc` or to `.git/config`) is never an entry (`wiki` v0.20.0 and later).
- **Nested bundles:** a folder holding its own `wiki.toml` is a separate bundle, and the bundle around it skips it (a later release of `wiki`). Its files stop being entries of the outer bundle, and a link into it is read like a link to an ignored file.
- **Links are relative** to the linking file in their canonical form, as `wiki` writes them since v0.7.0 (it was root-absolute). Obsidian's link format is *Relative path to file*.
- **Slugs are recommended, not required.** The rule is no spaces in a path, which `wiki tidy --slug` fixes in folder names too from a later release of `wiki`.
- **The checkbox list is a *task list*.** A *board* is now a `[[board]]` view.
- **The repos** are spec, wiki, skills and wiki-shell. The template repo plan is retired: `wiki init --shape` writes the starters.

### Fixed

- `skip` is `ignore`, the key `wiki` reads.
- Wikilinks are read by `wiki` for compatibility, flagged and converted, not silently dropped.
- The bundle-root `index.md` carries `okf_version`, and `wiki check` reads `log.md` (its date headings).
- `.wiki/` is reserved for a cache, not a cache that exists.
- Commands and flags that no longer exist: `wiki tasks` is `wiki checkboxes`, and `--type`/`--tag` are `--where`.
