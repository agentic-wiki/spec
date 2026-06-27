---
type: task
title: Skill repo
status: in-progress
priority: medium
tags: [skills, tooling]
---

Build `agentic-wiki/skills`: the operating manuals an agent follows to drive `wiki`, surfaced per runtime. Two exist today, `agentic-wiki` (the KB lifecycle: capture / refine / promote / index / retrieve / maintain) and `wiki-tasks` (a task backlog); they currently live in `wiki/.claude/skills/`. Remaining: move them into their own repo and add runtimes. They call `wiki` directly.
