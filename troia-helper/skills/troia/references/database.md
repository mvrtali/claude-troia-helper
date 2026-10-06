# Database access

TROIA's SQL commands are TROIA commands, not SQL passed through. At runtime the
interpreter translates them into the dialect of the connected database (IASDB,
MSSQL, MySQL, Oracle, PostgreSQL, DB2). Write one portable statement; do not
branch on database vendor.

## SELECT

```
SELECT [ALL | DISTINCT] {selectlist}
	FROM {table} [, {othertables}]
	[INNER | OUTER JOIN {jointable} ON {onclause}]
	[WHERE {condition}]
	[ORDERBY {columns} [ASC | DESC]]
	[INTO {targettable}]
	[ROWFETCHSTART {start}]
	[ROWFETCHLIMIT {limit}]
	[WITHCONTROL {rowcount}]
	[WITHCACHE]
```

```
SELECT CLIENT, USERNAME, PWDVALIDITY
	FROM IASUSERS
	WHERE CLIENT = SYS_CLIENT AND USERNAME LIKE NAMEPREFIX
	ORDERBY PWDVALIDITY DESC
	INTO USERS;
```

- The result lands in a table variable. With `INTO`, that table's column model
  is replaced by the select list. Without `INTO`, a table variable named after
  the database table is created or overwritten (`IASUSERS_USERNAME`).
- TROIA variables in `WHERE` are bound automatically with the right type; no
  quoting or concatenation needed. Inside the statement use SQL operators
  (`=`, `AND`, `OR`, `LIKE`), not `==` / `&&`.
- `ORDERBY` (one word) is the traditional spelling; `ORDER BY` is also accepted
  by the newer interpreter.
- CANIAS tables are client-dependent: filter with `CLIENT = SYS_CLIENT`.
- `ROWFETCHSTART` / `ROWFETCHLIMIT` are offset and limit applied on the
  application server, not in the SQL sent to the database.
- `WITHCONTROL n` asks the user whether to continue when more than `n` rows
  would be fetched.
- After any database command the generated statement is in the `SQL` system
  variable - the quickest way to see what was really sent.
- `(INDEX=indexname)` after the table name forces an index; discouraged.

Portable SQL functions (translated per database): `CONCAT`, `SUBSTRING`,
`LEFT`, `LEN`, `YEAR`, `MONTH`, `QUARTER`, `WEEK`, `DAYOFMONTH`, `DAYOFYEAR`,
`HOUR`, `MINUTE`, `DATEPART`, `DATEDIFF`, `DATEADD`, `DATESUB`.

## Dynamic ("complex") SELECT

`@VAR` substitutes the text of a variable into the statement at runtime.

```
ITEMS = 'USERNAME, CREATEDBY, CREATEDAT';
TABLENAME = 'IASUSERS';
COND = 'CREATEDBY = PCREATEDBY';

SELECT @ITEMS
	FROM @TABLENAME
	WHERE @COND
	INTO T1;
```

To append a dynamic piece to an otherwise fixed `WHERE`, the predefined
variable `SYSADDITIONALCRITERIA` must be used:

```
SYSADDITIONALCRITERIA = 'AND CREATEDBY = PCREATEDBY';

SELECT USERNAME, CREATEDBY
	FROM IASUSERS
	WHERE CLIENT = SYS_CLIENT @SYSADDITIONALCRITERIA
	INTO T1;
```

- Variable names inside the substituted text are still bound as variables, so
  values never need to be concatenated into the string.
- Fixed and dynamic text cannot be mixed in the select list or the FROM
  clause; only `WHERE` allows it, and only through `SYSADDITIONALCRITERIA`.
- Dynamic statements are parsed at runtime instead of at convert time. Use
  them only when the structure really varies.

## Row-by-row fetching

For result sets too large to hold in memory:

```
SELECTLINE USERNAME, CREATEDBY
	FROM IASUSERS
	WHERE CLIENT = SYS_CLIENT
	INTO T1;

WHILE 1
BEGIN
	FETCH T1;
	IF SYS_STATUS THEN
		BREAK;
	ENDIF;
	/* T1 holds the current row */
ENDWHILE;
```

`SELECTLINE` has the same syntax as `SELECT` but does not fetch.
`FETCH T1;` loads one row; `FETCH T1 SIZE n;` loads a block (8.02.01+).

## Performance

- The cheapest query is the one not issued. Question whether a `SELECT` is
  needed before tuning it, and select only the columns used.
- A `SELECT` for header data inside a loop over item rows is the classic
  mistake: read the header once before the loop.
- `WITHCACHE` caches a statement's result for the current session and the
  current user interaction only (cleared when control returns to the client;
  `CLEARDBSELECTIONCACHE()` clears it manually). It is a refactoring shortcut,
  not a design; prefer removing the repeated query.

## Connections

A session has one default connection shared by all its open transactions, so
anything that changes connection state affects every open transaction.

```
MAKENEWCONNECTION {connectionname} {user} {passw} {dbserver} {dbname};
SETACTIVECONNECTION {connectionname};
SETACTIVECONNECTION DEFAULT;
CLOSECONNECTION {connectionname};
```

`{user}` and `{passw}` are ignored (kept for compatibility); `{dbserver}` and
`{dbname}` select an entry from the `[Databases]` section of the server
configuration.

```
CONN = 'ARCHIVE';
MAKENEWCONNECTION CONN XXX XXX DBSERVER1 ARCHIVEDB;

IF SYS_STATUS THEN
	RETURN SYS_STATUSERROR;
ENDIF;

SETACTIVECONNECTION CONN;
SELECT * FROM USERACCOUNTS INTO ACCOUNTS;
SETACTIVECONNECTION DEFAULT;

CLOSECONNECTION CONN;
```

Custom connections must be closed by the code that opened them, on every path,
after switching back to `DEFAULT`.

## Database transactions

`BEGINTRAN;`, `COMMITTRAN;` and `ROLLBACKTRAN;` act on the active connection,
and `SYS_INDBTRANSACTION` reports its state. With several connections, each
needs its own begin/commit while it is the active one.

Messages raised between `BEGINTRAN` and `COMMITTRAN` are answered
automatically and shown to the user in bulk afterwards (see platform.md).

## Not documented publicly

The chapter of the public book covering writes is an unfinished stub. It names
the topics - updating, inserting and deleting data, `EXECUTESQL`, the
persistency flags `DELETED` / `INSERTED` / `READ` / `UPDATED` / `CHANGED` /
`CHECKED`, table definition in ODBA, select/insert/update/delete user rights -
but gives no syntax. The docs also mention `INSERTSQL`, `UPDATESQL` and
`DELETESQL`, which only build the statement into `SQL` without executing it.

When a task needs a write, do not invent the syntax. Find an existing
`INSERT` / `UPDATE` / `DELETE` in the user's code base and follow it, or ask
the user for an example, and flag the generated statement as needing a check.
