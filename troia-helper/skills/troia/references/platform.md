# Platform: classes, dialogs, transactions, messages, inheritance

## Item types

| Item | What it is |
| --- | --- |
| Class | members + methods; in practice mostly a set of methods, with data kept in tables |
| Dialog | UI form: controls, events and methods. Dialogs are not classes |
| Component | reusable group of controls |
| Report | dialog-like item rendered to PDF, text or printer |
| Transaction | a database record that makes a standalone application: id, caption, start dialog |

Development flow: every change requires a hotline (change request). "Convert"
parses and compiles an item (it fails on parse errors); "Save" writes the
binary with texts to `.dlg` (dialogs, reports, components; one per language)
or `.cls`. At runtime only those files are read, never the development tables.

`{userfilepath}\jdlg\{module}\{languagecode}{dialog}.dlg` - for example
SALT01D001 in English is `jdlg\SAL\ET01D001.dlg`.

Development tables (useful when searching code through SQL)

| Table | Holds |
| --- | --- |
| `SYSCLSHEAD` | class header: name, base class |
| `SYSCLSFUNC` | class methods and their code |
| `SYSDIALOGS` | dialog header |
| `SYSDLGCODES` | events and methods of dialogs and their controls |
| `SYSDLGTEXTS`, `SYSDLGFUNCTEXTS` | dialog and method captions per language |
| `SYSCONTROLS`, `SYSCTLTEXTS` | controls and their captions |
| `SYSTRANS`, `SYSTRANSTXT` | transactions |
| `SYSCLSREF`, `SYSDLGREF` | system crosses |
| `IASUSERCLSREF`, `IASUSERDLGREF` | user/profile crosses |

Tools: TROIA IDE (permission `DEVELOPMENT`, or `DEVELOPMENT(READ-ONLY)`),
SYST00 transactions, SYST01 locks, SYST02 messages, SYST03 users, SYST06 system
parameters, DEVT00 class browser, DEVT01 database browser (ODBA), DEVT06
hotlines, DEVT07 search in code, DEVT08/DEVT09 class/dialog dynamic link
(cross), DEVT30 run-code test (DEVT11 before 9.03), DEVT31 traces, DEVT40
execute SQL.

Debugging is normally done with traces: a runtime switch (permission `TRACE`)
logs every executed line with values; no debug build is needed and it works in
production. Trace lines look like
`[DEVT11D001.RUNBUTTON.2 4] : RESULT = INSTANCE.M1(10); [11]`.

CANIAS versions map to platform builds: 602 = 3.08, 603 = 5.01, 604 = 5.02,
802 = 8.02, 803 = 8.03. Builds from 23.02.10-01 support 8.02, 8.03 and 9.03.

## Classes

```
/* method code */
PARAMETERS:
	INTEGER PA,
	INTEGER PB;

LOCAL:
	INTEGER MAXNUM;

MAXNUM = THIS.MAX(PA, PB);
RETURN 'Maximum is ' + MAXNUM;
```

- A method body is its code; the name and return type are set in the IDE.
  Parameters are declared with `PARAMETERS:` as the first command.
- `_VARIABLES` and `_CONSTRUCTOR` run automatically when an instance is
  defined. Convention: members in `_VARIABLES`, initial state in
  `_CONSTRUCTOR`. They run once per instance even if the definition command
  executes again.
- Defining an instance: `OBJECT: MATHTEST REC;` (or LOCAL/GLOBAL/MEMBER).
- Calling: `REC.SUM(5, 6)`; inside the class `THIS.SUM(5, 6)`. Recursion works.
- Members: `REC@FACTOR = 2;` All members are public. `@` cannot be chained
  (`A@B@C` is invalid).

## Dialogs

Dialog events, in opening order

| Event | When |
| --- | --- |
| `BEFORE` | first; controls exist, dialog not visible |
| `AFTER` | after BEFORE; still not visible |
| `TRANSCALLED` | start dialog only, after AFTER, when the transaction was called with input parameters |
| `ONSHOW` | when the dialog is visible |
| `ONTIMER` | periodically, once `SETTIMER {ms};` is set (`SETTIMER 0;` stops it) |

`BEFOREEXTENSION` also exists. Events can be called like methods:
`THIS.AFTER();`.

Controls have a type and subtype (stored in `SYSCONTROLS`) and define a global
control symbol of the same name. TextField subtypes and symbol types: Text,
Editor, File Chooser, Troia Editor, Password (STRING); Decimal (DECIMAL); Long
(LONG); Integer (INTEGER); Date (DATE); Datetime (DATETIME); plus Color, Money,
Quantity, Percent, Factor, Duration, HTML, Link, Phone, Rich Editor, Time,
Times and others. TextField events: GainFocus, LoseFocus, TextChanged (fires
before LoseFocus), ZoomBefore/ZoomAfter, Drag/Drop, RightClickMenu.

Dialog methods can be shown as right-click menu items ("Show on Menu") and
bound to a control so the item follows that control's enabled/visible state.

Navigation
```
CALL DIALOG {dialog};
CALL DIALOG WITH LOCATION {x}, {y} SIZE {width}, {height};
SHUTDOWN;
```
- `CALL DIALOG` suspends the calling method until the opened dialog closes,
  then continues with the next command.
- `SHUTDOWN;` closes the most recently opened dialog; closing the last dialog
  ends the transaction.
- No parameter passing is needed between dialogs: globals and control symbols
  of the calling dialog are visible in the called one, and remain defined after
  a dialog closes. Name clashes between dialogs are therefore real bugs.

## Transactions

A transaction is the only thing that can execute TROIA code and is the widest
variable scope. Defined in SYST00 with id, caption, start dialog and module.

```
CALL TRANSACTION {tran} [({inputs})];
CALL TRANSACTION {tran} [({inputs})] WITH WAIT [({outputs})];
CALL TRANSACTION {tran} ({inputs}) INSERVER [({outputs})];
```

- Plain form: does not stop the caller; the transaction opens after the current
  server activity ends. No output parameters.
- `WITH WAIT`: caller stops until the user closes the called transaction;
  outputs are then available.
- `INSERVER`: the server opens the transaction and its start dialog, runs the
  events and closes it, with no UI.
- Inputs become globals of the same name, value and type in the called
  transaction, so pass variables rather than constants.
- `SYS_CALLEDTRANSACTIONID` holds the id of the transaction just opened (build
  26.10.02-01+); read it immediately after the call.

Batch transactions: a transaction can carry "Batch Code" (SYST00) that runs
after its start dialog opens. Scheduling is done by the operating system (Task
Scheduler, cron); the platform has no scheduler.

Transaction info: `SYS_TRANSACTION`, `SYS_TRANSACTIONID`,
`SYS_TRANSACTIONTYPE`, `SYS_ISBATCHTRANSACTION`,
`SYS_ISSERVERONLYTRANSACTION`, `SYS_TRANCREATEDBY`, `SYS_TRANCREATEDAT`,
`SYS_TRANMODIFIEDBY`, `SYS_TRANMODIFIEDAT`.

## Messages

Message texts are never written in code. They are defined per language in
SYST02 and referenced by module, type and id:

```
MESSAGE {module} {type}{id} WITH {parameters};

MESSAGE BAS C100 WITH;

IF CONFIRM == 'YES' THEN
	...
ENDIF;
```

Types: `I` information, `E` error, `W` warning, `C` confirmation (yes/no),
`O` option, `P` parameter (keyboard input). Texts may contain `%s`
placeholders (or `%s1`, `%s2` when word order differs by language) filled from
the values after `WITH`. `WITH` is written even when there are none.

Results
- `CONFIRM`: `'YES'` / `'NO'` for confirmations; the ordinal of the chosen
  option for option messages.
- `SYS_CONFIRMTEXT`: text of the chosen option, or the typed input.
- `SYSMESSAGE`: the final message text.

Without a user
- A message normally interrupts server execution to wait for the client ("code
  breaking").
- In batch and INSERVER execution, messages are answered with the default
  option, confirmations with YES, information and warnings are skipped, and
  everything is logged to table `IASBATCHERR` (filter by
  `CLIENTCONNID = SYS_CLIENTCONNECTIONID AND TRANSID = SYS_TRANSACTIONID`).
  `SETBATCHTRAN TRUE;` / `SETBATCHTRAN FALSE;` switch that mode in code.
- Inside a database transaction, messages are answered automatically and
  delivered to the client together after the transaction ends; they are
  collected in the system table variable `SYSBATCHMESSAGES`.
  `SETSERVERONLY(1)` / `SETSERVERONLY(0)` switch server-only mode.

Also available: `PUSHNOTIFICATION` (build 24.12.26-01+), `ATTENTION`, `ALERT`.
Their syntax is not in the public docs.

## Inheritance

Classes, dialogs, reports and components support single inheritance (chains
allowed). It is resolved at compile time. A child method with the same name as
a base method overrides it; TROIA developers call this "inheriting a method".
On dialogs, controls and control events can be overridden independently.

`SUPER();` - a compile-time include, not a call
```
/* base M1 */
SELECT * FROM IASUSERS INTO T1;
RETURN;

/* child M1 */
SUPER();
CLEAR ALL T1;      /* never runs: the base RETURN was pasted above it */
```
- The base code is pasted at the position of `SUPER();`. Any `RETURN` in the
  base ends the child method too, and no return value can be captured.
- Works in classes, dialogs, components and reports (classes from 5.01.07 /
  5.02.03).
- `_CONSTRUCTOR` and `_VARIABLES` always include the base code automatically;
  do not write `SUPER();` there.

`SUPER.METHOD(args)` - a real call
```
R = SUPER.M1(PARAM);
R = R + 1;
RETURN R;
```
Classes only (5.01.07 / 5.02.03 and later). Returns the base result and is
unaffected by the base method's `RETURN`.

## Cross

A cross tells the loader to use item B wherever item A is requested. It is a
database record applied at runtime, so standard code is customised without
being modified - typically a cross from a standard class or dialog to a child
that overrides parts of it. This is the normal way customer-specific changes
are made in CANIAS.

- System crosses apply to everyone: DEVT08 for classes (`SYSCLSREF`), DEVT09
  for dialogs, reports and components (`SYSDLGREF`).
- User and profile crosses: SYST03, "Class Reference" / "Dialog Reference" tabs
  (`IASUSERCLSREF`, `IASUSERDLGREF`).
- Precedence: the user's own crosses, then profile crosses (deepest profile
  first), then system crosses.
- Crosses chain (A to B, B to C). A cycle is an error that can hang login.
  `A -> A` at a more specific level cancels an inherited cross.
- Crosses are loaded at login; changes take effect at the next login.

When explaining why code "does not run", check whether a cross redirects the
class or dialog to a different item for that user.

## Files and external programs (brief)

```
OPEN FILE PATH FORNEW;
PUT '<b>text</b>';
CLOSE FILE;
DELETEFILE PATH;
FILELIST FOLDER TO T1;          /* rows with a NAME column */

RUNFILE PATH;                                         /* open with the client's default app */
RUNPROGRAM COMMAND [WITH WAIT [INTO RESULT]];
```
A path starting with `*` refers to the client machine; otherwise it is on the
application server. `RUNFILE` only works for client paths. Fetch the `58File`
chapter for the full file API.
