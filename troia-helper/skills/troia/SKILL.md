---
name: troia
description: TROIA language knowledge for CANIAS ERP (IAS). Use whenever reading, writing, reviewing, explaining or debugging TROIA code - dialogs, classes, reports, transactions, table variables, TROIA SELECT / LOOP / LOCATERECORD statements, SYS* development tables, traces - or when porting TROIA logic to another stack.
---

# TROIA (CANIAS ERP)

TROIA is a command-based 4GL that runs on the TROIA Platform (JVM). Code lives in
the database (dialogs, classes, reports, components), is edited in TROIA IDE under
a hotline (change request), and is compiled by Convert/Save into `.dlg` / `.cls`
files. It looks a little like ABAP and a little like SQL, and most habits from
C-family languages produce code that converts but behaves wrongly.

Read the reference file for the area you are working in before writing code:

| Working on | Read |
| --- | --- |
| Types, variables, scope, operators, flow control, strings, dates | [references/language.md](references/language.md) |
| TABLE variables: cells, active row, flags, LOOP, LOCATERECORD, SORT, copy/merge | [references/tables.md](references/tables.md) |
| SELECT, SELECTLINE/FETCH, dynamic SQL, connections, DB transactions | [references/database.md](references/database.md) |
| Classes, dialogs, events, transactions, messages, inheritance, cross, dev tools | [references/platform.md](references/platform.md) |

## Shape of TROIA code

```
/* class or dialog method */
PARAMETERS:
	STRING PCREATEDBY;

LOCAL:
	TABLE USERS,
	INTEGER MAXVALIDITY,
	STRINGBUILDER SB;

MAXVALIDITY = THIS.CALCMAXVALIDITY() * 4;

SELECT USERNAME, CREATEDBY, PWDVALIDITY
	FROM IASUSERS
	WHERE CLIENT = SYS_CLIENT AND CREATEDBY = PCREATEDBY
	ORDERBY USERNAME
	INTO USERS;

LOOP AT USERS CRITERIA COLUMNS PWDVALIDITY VALUES MAXVALIDITY
BEGIN
	APPENDSTRING USERS_USERNAME TO SB;
	APPENDSTRING TOCHAR(10) TO SB;
ENDLOOP;

IF USERS_ROWCOUNT == 0 THEN
	RETURN '';
ENDIF;

RETURN SB;
```

## Rules that are easy to get wrong

Syntax
- The only comment form is `/* ... */`. No `//`, `#` or `--`.
- Statements end with `;`. Code and identifiers are written in UPPER CASE by convention.
- Strings use single quotes and have no escape sequences. Build special
  characters with `TOCHAR(39)` (quote) and `TOCHAR(10)` (newline).
- Blocks are keyword pairs, never braces: `IF .. THEN .. ELSE .. ENDIF;`,
  `WHILE .. BEGIN .. ENDWHILE;`, `LOOP AT .. BEGIN .. ENDLOOP;`,
  `PARSE .. BEGIN .. ENDPARSE;`, `SWITCH .. ENDSWITCH;`.
- There is no `ELSE IF`, `FOR` or `FOREACH`. Nest `IF` inside `ELSE`; iterate
  tables with `LOOP AT`.
- `SWITCH` takes a variable only (not an expression), compares as strings, and
  has no `BREAK` between cases.

Expressions
- TROIA expressions use `==`, `!=`, `&&`, `||`, `!`. Inside SQL commands the
  operators are SQL's: `=`, `AND`, `OR`, `NOT`. Mixing the two sets is the most
  common mistake when generating TROIA.
- `&&` and `||` do not short-circuit; both operands are always evaluated. Do
  not guard a call with the left operand - nest `IF`s instead.
- Integer / integer is integer division (`1 / 2` is `0`). Make one operand
  DECIMAL (`1.0 / 2`).
- `MOVE x TO y;` accepts no expression on the source side. Use `=` for anything
  computed.
- Decimal separator in code is always `.`; hard-coded dates are always
  `'DD.MM.YYYY'` or `'DD.MM.YYYY HH:MM:SS'`, regardless of user locale.

Names and scope
- An undefined variable is not an error: it evaluates to its own name. A typo
  in a variable name converts cleanly and silently yields a wrong value, so
  check every identifier against its definition.
- A call without a receiver is a system function. Calling a method of the same
  class/dialog requires `THIS.METHOD()`; other instances use `INSTANCE.METHOD()`.
- Class members are reached with `@` (`REC@FIELD`), one level only. Table cells
  are reached with `_` (`T1_USERNAME`, `T1[3]_USERNAME`). Table row indexes
  start at 1.
- `OBJECT:` picks scope by context (see language.md). Prefer `LOCAL:`,
  `MEMBER:` and `GLOBAL:` in new code; expect `OBJECT:` everywhere in existing
  code. Tables and class instances defined with `OBJECT:` are always global.
- Dialog controls create global variables with the control's name, and globals
  outlive the dialog that defined them within the transaction.

Runtime behaviour
- Commands report failure through `SYS_STATUS` (0 = ok) and `SYS_STATUSERROR`
  rather than exceptions. Check it after `LOCATERECORD`, `FETCH`,
  `MAKENEWCONNECTION` and similar commands.
- `SELECT` without `INTO` creates/overwrites a table variable named after the
  database table. Always write `INTO`.
- `LOOP AT` moves the table's active row; it is not restored afterwards.
- `CALL DIALOG` blocks until the dialog closes. `CALL TRANSACTION` does not
  block unless `WITH WAIT` or `INSERVER` is used.
- `SUPER()` is a compile-time paste of the base method, not a call: a `RETURN`
  in the base method ends the overriding method too. `SUPER.METHOD()` (classes
  only) is a real call.
- `MESSAGE` inside `BEGINTRAN .. COMMITTRAN`, batch or INSERVER code is
  answered automatically (confirmations get YES); never rely on the user's
  answer there.

Performance
- Loop tables with `LOOP AT`, not `WHILE` plus an index.
- Prefer `CRITERIA COLUMNS` (evaluated in the Java layer) to `WHERE`; build a
  hash index when the same columns are probed repeatedly.
- Compute loop-invariant values before the loop, not in the `WHERE` condition.
- Do not put a `SELECT` for header data inside a loop over item rows.
- Use `STRINGBUILDER` + `APPENDSTRING` for string building in loops.

## What the public documentation does not cover

These reference files are distilled from the public book "Programming with
TROIA". Several chapters of that book are still stubs, so the following have
no documented syntax here:

- writing to the database (`INSERT`, `UPDATE`, `DELETE`, `EXECUTESQL`) and the
  full persistency-flag workflow
- exception handling and debugging
- table control details (aggregates, tree tables, filtering, conditional format)
- reports and printing, barcodes, mailing, SMS, digital signing

For these, and for any command or system function whose exact parameters are
not in the references, do not guess. Look for an existing usage in the user's
own TROIA code, or tell the user it needs checking in TROIA Help. State
clearly which parts of generated code are unverified.

Behaviour also varies by platform build (3.08 / 5.01 / 5.02 / 8.02 / 8.03 /
9.03). Features the docs tie to a version are marked in the references; ask
which CANIAS version the user targets when it matters.

## Topics available upstream but not distilled

Fetch the chapter when the task needs it. Sources are
`https://raw.githubusercontent.com/bahtiyartan/troia/master/docs/<file>.rst`:

`02Platformbasics`, `26Domains`, `51Theme`, `58File`, `62Components`, `66FTP`,
`68Multithreading`, `70WebServices`, `72RestfulWebServices`, `73Endpoints`,
`74VectorDB`, `75LLMSupport`, `76XMLjson`, `78HTTPOperations`, `82Port`,
`92ProgramsLibraries`, `93ClientPlugin_`, `93MobileToolkit`, `94Java`,
`96JMXMonitoring`, `97VisualVM`.

Rendered book: https://troia.readthedocs.io/en/latest/index.html
