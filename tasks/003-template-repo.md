---
type: task
title: Template repo
status: in-progress
priority: medium
tags: [template, tooling]
---

Create the `agentic-wiki/template` repo, shaped as a **registry of named KB templates** (e.g. `templates/personal/`, `templates/corporate/`) rather than a single bundle, so the CLI can embed the set and offer `wiki init --template <name>`. Consumed via `go:embed` as a Go-module dependency (no submodule); embed the template dirs only so the repo's `.git` is never bundled; ship each `gitignore` (renamed to `.gitignore` on write). The skill set for `wiki skills install` is embedded the same way from the skills repo. See the CLI's scaffold-registry task.
