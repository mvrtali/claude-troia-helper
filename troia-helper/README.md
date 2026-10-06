# TroiaHelper

A Claude Code plugin that teaches Claude the TROIA programming language used
by CANIAS ERP, so it reads, writes and reviews TROIA code correctly.

## Contents

| Path | Purpose |
| --- | --- |
| `skills/troia/SKILL.md` | Core rules and pitfalls; loaded automatically when TROIA work comes up |
| `skills/troia/references/language.md` | Types, scope, operators, flow control, strings, dates |
| `skills/troia/references/tables.md` | TABLE variables: flags, LOOP, LOCATERECORD, SORT, copy/merge |
| `skills/troia/references/database.md` | SELECT, dynamic SQL, fetching, connections, DB transactions |
| `skills/troia/references/platform.md` | Classes, dialogs, transactions, messages, inheritance, cross |

## Install

From Claude Code:

```
/plugin marketplace add mvrtali/claude-troia-helper
/plugin install troia-helper@mavera-plugins
```

The skill triggers on its own when a task involves TROIA, and can be invoked
directly with `/troia-helper:troia`.

## Extending

The knowledge comes from the public book "Programming with TROIA"
(https://troia.readthedocs.io, source https://github.com/bahtiyartan/troia),
rewritten as a compact reference. Some chapters of that book are unfinished -
notably database writes (INSERT / UPDATE / DELETE), exception handling and the
table control - so those areas are marked as undocumented in the skill.

To fill the gaps, add your own verified notes and code samples to the files in
`skills/troia/references/`, or add a new reference file and link it from the
table at the top of `SKILL.md`. Bump `version` in
`.claude-plugin/plugin.json` after changes so installed copies update.
