# TABLE variables

TABLE is the central type of TROIA: an in-memory, two-dimensional structure
that resembles a database table. Entities are normally held in tables rather
than in class members. "Table" here means the variable, not a database table
and not the table control on a dialog - although every table control has a
global table variable of the same name as its model, so everything below
applies to UI tables too.

## Creating structure and rows

```
LOCAL:
	TABLE T1;

APPEND COLUMN CODE, STRING, 100 TO T1;
APPEND COLUMN QTY, INTEGER, 10 TO T1;

APPEND ROW TO T1;
T1_CODE = 'A-1';
T1_QTY = 5;
```

- `APPEND COLUMN {name}, {type}, {length} TO {table};` Column types: STRING,
  INTEGER, TEXT, DATE, DATETIME, DECIMAL, LONG, TIME, TIMES. Names are unique;
  TABLE and VECTOR columns are not allowed.
- `APPEND ROW TO {table} [ATHEAD | ATBETWEEN];` The new row becomes the active
  row.
- `SELECT ... INTO T1;` replaces the column model with the select list and
  fills the rows. `WHERE 1 = 2` is the idiom for "give me the structure only".
- `CONSTRUCT` also builds a table definition; rarely used.

## Reading and writing cells

| Form | Meaning |
| --- | --- |
| `T1_COLNAME` | cell on the active row |
| `T1[ROWINDEX]_COLNAME` | cell on a specific row; indexes start at 1 |

The active row is an internal cursor. It is moved by the UI, by `LOOP`,
`LOCATERECORD`, `APPEND ROW`, and explicitly:

```
T1_ACTIVEROW = 3;
READ T1 WITH INDEX 3;
READ T1 WITH FIRST;        /* also LAST, NEXT, PREV */
```

Active row is a single row and is not the same thing as a selected row.

## Flags

Flags are reserved names read with `_` like columns; they cannot be used as
column names.

Table-level

| Flag | Type | Writable | Meaning |
| --- | --- | --- | --- |
| `ACTIVEROW` | INTEGER | yes | active row index, 1..row count (0 when empty) |
| `ROWCOUNT` | INTEGER | no | number of rows |
| `DBTABLENAME` | STRING | no | database table the data came from |
| `HASSELECTEDROW` | INTEGER | no | any row selected |
| `ACTIVECOL`, `ACTIVECOLNAME` | | no | active column of a UI table |
| `ARROWSTATE` | INTEGER | no | for the ArrowClick event |

Row-level (active row, or `T1[n]_FLAG`)

| Flag | Meaning |
| --- | --- |
| `SELECTED` | 1 when the user selected the row |
| `HIDE` | 1 hides the row in the UI |
| `BKCOLOR` | row colour |
| `ROWTOOLTIP` / `FYI` | tooltip text |
| `FILTERED` | 1 when hidden by a UI filter |
| `SUMMARYROW` | set to 1 on subtotal rows so they are not double counted |
| `CHECKED` | free for application use |

Persistency flags (INTEGER 0/1, writable, maintained automatically when data
is read from the database or a cell changes)

| Flag | Meaning |
| --- | --- |
| `READ` | row came from the database |
| `INSERTED` | new row |
| `UPDATED` | changed after being read |
| `DELETED` | deleted by user or code |

These flags are how TROIA decides which statement a row needs when a table is
written back. The write-back commands themselves are not documented publicly;
see database.md.

Helpers: `SELECTEDROWCOUNT(T1)`, `GETCOLUMNCOUNT(T1)`.

## Looping

```
LOOP AT T1 [WHERE {condition}]
BEGIN
	...
ENDLOOP;

LOOP AT T1 CRITERIA COLUMNS {col1,col2} VALUES {v1,v2} [NOTCASESENSITIVE]
BEGIN
	...
ENDLOOP;

BUILDHASHINDEX {indexname} COLUMNS {cols} ON T1 [FORCE];
LOOP AT T1 CRITERIA INDEXED INDEX {indexname} VALUES {v1,v2}
BEGIN
	...
ENDLOOP;
```

- Every variant advances the active row; it is left changed after the loop.
- `WHERE` evaluates a TROIA expression per row in the interpreter (slowest).
- `CRITERIA COLUMNS` matches by equality in the Java layer (faster). Row flags
  can be used as columns: `CRITERIA COLUMNS CREATEDBY,INSERTED VALUES USR,1`.
- A hash index is computed once and reused; best when the same columns are
  probed many times with different values. `BUILDHASHINDEX` does nothing if the
  index name exists unless `FORCE` is given. `HASHASHINDEX()` tests existence.
- Do not loop tables with `WHILE` and an index.
- Hoist loop-invariant expressions out of the `WHERE` condition into a variable
  before the loop.

## Finding one row

```
LOCATERECORD SEQUENTIAL COLUMNS {cols} VALUES {vals} ON T1 [NOTCASESENSITIVE] [NEXT] [LAST];
LOCATERECORD BINARYSEARCH COLUMNS {cols} VALUES {vals} ON T1 [NOTCASESENSITIVE];
LOCATERECORD INDEXED INDEX {indexname} VALUES {vals} ON T1 [NEXT] [LAST];
```

On success the active row moves to the match. On failure the active row is
unchanged and `SYS_STATUS` is 1, so always test it:

```
LOCATERECORD SEQUENTIAL COLUMNS CODE VALUES PCODE ON T1;
IF SYS_STATUS THEN
	RETURN 0;
ENDIF;
```

`BINARYSEARCH` requires the table to be sorted ascending on the given columns.

## Sorting

```
SORT T1 [CASESENSITIVE] ON CREATEDBY, DESC CREATEDAT;
SORT T1 ON @SORTSPEC;        /* SORTSPEC = 'CREATEDBY, DESC CREATEDAT' */

SORT T1 HIERARCHICAL IDCOLUMN 'NAME' PARENTIDCOLUMN 'COUNTRY'
	[ROOTINDICATOR {value}] [MARKLEAFSASNODE];
```

Default is ascending and case-insensitive. Hierarchical sort needs unique ids
and single id / parent-id columns. A sort cannot be undone. After a user sorts
a UI table, the ColumnSort event fires with `SYS_SORTEDTABLE`,
`SYS_SORTEDCOLUMN` and `SYS_SORTED` set (the last one in `SORT` syntax).

## Removing rows and columns

```
CLEAR ROW T1;        /* removes the active row; cursor stays on the same index */
CLEAR ALL T1;        /* removes all rows; ACTIVEROW becomes 0 */

CLEARTABLE T1 WHERE T1_CREATEDBY == 'BTAN';
CLEARTABLE T1 CRITERIA COLUMNS CREATEDBY VALUES 'BTAN' [NOTCASESENSITIVE];

DESTROYTABLE T1;                              /* drops all columns and data */
REMOVE COLUMN CREATEDBY FROM T1;              /* marks removed: skipped in DB statements */
REMOVE COLUMN CREATEDAT PERMANENT FROM T1;    /* physically removed */
REMOVE COLUMN @COLVAR FROM T1;                /* name held in a variable */
```

## Copying between tables

```
COPY TABLE SRC INTO DST [WITHFLAGS];          /* structure and data */
COPY STRUCTURE SRC INTO META;                 /* COLNAME, COLTYPE, COLLEN, COLPRE, COLNOT */

MOVE-CORRESPONDING SRC TO DST [WITHFLAGS];    /* same-named columns, active row to active row */

MERGETABLE SRC INTO DST [WITHFLAGS] WHERE {condition};
MERGETABLE SRC INTO DST [WITHFLAGS] CRITERIA COLUMNS {cols} VALUES {vals} [NOTCASESENSITIVE];
MERGETABLE SRC INTO DST [WITHFLAGS] CRITERIA INDEXED INDEX {indexname} VALUES {vals};
```

- `MOVE-CORRESPONDING` works on active rows: `APPEND ROW TO DST;` first when
  adding. It also copies CREATEDBY/CREATEDAT/CHANGEDBY/CHANGEDAT, except when
  the source is configured as a check table.
- `WITHFLAGS` carries CHECKED, DELETED, INSERTED, READ, UPDATED, ROWTOOLTIP and
  SUMMARYROW. HIDE and SELECTED are never carried.
- `MERGETABLE` replaces the common "loop, test, append, move-corresponding"
  pattern.

## Showing a table variable on a dialog

The docs repeatedly use `SET TMPTABLE TO TABLE TMPTABLE;` after filling a
table to push it to the table control of that name, without documenting the
command further. Follow the pattern used in the surrounding code.

## Table control events

RowInsert, RowDelete, RowUndelete, CellChangeBefore, CellChangeAfter,
ZoomBefore, ZoomAfter, ColumnDoubleClick, ColumnSort, Copy, Paste, Drag, Drop,
RightClickMenu, ExpandBefore, ArrowClick, ButtonCellClick. Their parameters and
firing order are not described in the public docs.
