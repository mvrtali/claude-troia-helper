# TroiaHelper for Claude Code

A free Claude Code plugin that teaches Claude the TROIA programming language
used by CANIAS ERP, so it reads, writes and reviews TROIA code correctly.

## Install

In Claude Code:

```
/plugin marketplace add mvrtali/claude-troia-helper
/plugin install troia-helper@mavera-plugins
```

The skill loads on its own when a task involves TROIA. You can also call it
directly with `/troia-helper:troia`.

## What it knows

- Language core: types, scope, operators, flow control, strings, dates
- TABLE variables: flags, `LOOP`, `LOCATERECORD`, `SORT`, copy and merge
- Database access: `SELECT`, dynamic SQL, row-by-row fetching, connections,
  database transactions
- Platform: classes, dialogs, transactions, messages, inheritance, cross

See [troia-helper/README.md](troia-helper/README.md) for the file layout and
how to extend it.

## Known gaps

The public documentation this plugin is based on does not yet cover database
writes (`INSERT` / `UPDATE` / `DELETE` / `EXECUTESQL`), exception handling,
the table control, or reports. The skill tells Claude not to guess in those
areas. Contributions with verified examples are welcome.

## Disclaimer and credits

This is an unofficial community project. It is not affiliated with, endorsed
by or supported by IAS / CANIAS. TROIA and CANIAS are names of their
respective owner.

The reference material is an independent summary based on the public book
"Programming with TROIA" by Bahtiyar Tan
([troia.readthedocs.io](https://troia.readthedocs.io),
[source](https://github.com/bahtiyartan/troia)). For authoritative and
complete information, use that book and the TROIA Help of your installation.

## License

MIT - see [LICENSE](LICENSE).
