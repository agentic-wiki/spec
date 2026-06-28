---
type: task
title: "Skill audit: substantive issues in the agentic-wiki and agentic-backlog skills"
status: done
priority: high
tags: [skills, audit, docs]
---

# Skill Audit

Audited both SKILL.md files for issues that would actually trip up an agent. The substantive issues below should be resolved in the agentic-wiki and agentic-backlog skills.

| Issue | Why it mattered | Resolution |
|---|---|---|
| `index.md` frontmatter contradiction | Spec required `type` on every file, but reserved files carry none | Reserved files (`index.md`, `log.md`) carry no frontmatter and are exempt from `type`; the root `index.md` may carry `okf_version`. |
| Exit-code semantics | Agent treated exit 1 (no match) as a failure | Documented 0 found, 1 none, 2 error. Follow-up: refining this to the ls/grep/lint convention is now tracked as a task in the wiki repo's backlog. |
| Warnings vs staying green | Agent might block commits on informational warnings | Both skills state `wiki check` warnings are informational and never block. |
| `resource:` vs `source:` | A live-data pointer was conflated with provenance | agentic-wiki distinguishes `resource:` (the live thing), `source` (provenance), and ordinary body links. |
| Search semantics | Agent assumed search was a structured field query | Documented: case-insensitive substring over frontmatter and body; use `list --type/--tag` for field queries. |
| Division of labor | Unclear what the agent does vs the tool | Both skills: the agent reads, writes, and edits Markdown; `wiki` is the deterministic engine. |
| Merge conflicts | No guidance for git conflicts | Both skills: preserve both sides, merge frontmatter sensibly, escalate only genuine judgment calls. |

All seven are live in the skills, which are the source of truth for skill conventions.
