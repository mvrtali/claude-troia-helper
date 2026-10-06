# TROIA language core

## Data types

| Type | Notes | Default |
| --- | --- | --- |
| `INTEGER` | | 0 |
| `LONG` | | 0 |
| `DECIMAL` | all floating point numbers | 0.0 |
| `BOOLEAN` | 0 or 1; 5.01+. Older code uses `INTEGER` 0/1 | 0 |
| `STRING` | no fixed maximum length; also serves as "char" | empty |
| `STRINGBUILDER` | only for concatenation with `APPENDSTRING` | empty |
| `DATE` | `23.04.1920` | time of definition |
| `DATETIME` | `29.10.1923 16:30:30` | time of definition |
| `TIME` | `21:30` | |
| `TIMES` | `21:30:13`; 5.01+ | |
| `TABLE` | two-dimensional in-memory structure, see tables.md | |
| `VECTOR` | one-dimensional, mixed element types | |

There are no byte/char/short/float types. A class name is used as a type to
define an instance (`MYCLASS REC`).

Command, function and system-variable names are reserved; do not use them as
identifiers. Avoid names starting with `SYS`, and do not name variables `AND`,
`OR` or `NOT`.

## Defining variables

```
LOCAL:
	STRING NAME,
	INTEGER COUNTER,
	TABLE ITEMS,
	MYCLASS REC;
```

`GLOBAL:`, `MEMBER:` and `OBJECT:` share this form: comma-separated
`TYPE NAME` pairs, closed by `;`. Mixed types are allowed in one command.

Scopes
- Global: visible to every dialog and class running in the same transaction.
- Member: per class instance; only definable in class code. Accessed from
  outside as `INSTANCE@NAME`.
- Local: the whole method/event. There is no block scope and no static locals.
  Each call (including recursive calls) gets its own locals.
- Narrower scope shadows wider: local over member over global.
- Method parameters are locals, declared with `PARAMETERS:` at the top of the
  method.

`OBJECT:` chooses the scope from context:

| Defined type | Dialog/report event or method | Class `_CONSTRUCTOR` / `_VARIABLES` | Regular class method |
| --- | --- | --- | --- |
| TABLE | global | global | global |
| class instance | global | global | global |
| simple types | global | member | local |

Other ways variables come into existence
- Every dialog control defines a global "control symbol" with the control's
  name; its type follows the control subtype.
- `SELECT ... INTO X` defines table `X` if it does not exist.

Behaviour to know
- Using an undefined variable is not a compile error; it yields its own name
  as value. `GETVARTYPE(X)` returns `'UNKNOWN TYPE'` for it.
- Re-defining an existing variable with a second, different definition command
  is ignored (value kept).
- Re-executing the same definition command (in a loop, or an event fired
  again) re-initialises the variable to its default.
- A definition for a name that is already a control symbol is ignored.
- `GETVARTYPE(var)` returns the type name as a string.

Common system variables (mostly read-only):

| Variable | Meaning |
| --- | --- |
| `SYS_CLIENT`, `SYS_LANGU`, `SYS_USER` | login client, language, user |
| `SYS_CURRENTDATE` | current datetime, re-read on every access |
| `SYS_VERSION` | platform server version |
| `SYS_CURRENTDIALOG` | name of the current dialog |
| `SYS_STATUS`, `SYS_STATUSERROR` | status / error text of the last command |
| `SYS_AFFECTEDROWCOUNT` | rows affected by last insert/update/delete |
| `SQL` | last SQL text sent to the database |
| `CONFIRM`, `SYS_CONFIRMTEXT`, `SYSMESSAGE` | results of `MESSAGE` |
| `SYS_MINDATE`, `SYS_MAXDATE` | configured date limits |

## Operators

- Arithmetic: `+` (also string concatenation), `-`, `*`, `/`, `%`, `^` (power).
- Relational: `==`, `!=`, `>`, `>=`, `<`, `<=`.
- Logical: `!`, `&&`, `||`. No short-circuit evaluation.
- `AND`, `OR`, `NOT` exist only inside SQL commands.

Assignment
```
MOVE 'Hello' TO S1;          /* literal or variable only */
S1 = S1 + ' world';          /* expressions need = */
N = 3 * (5 + N);
```

Expressions are accepted as method/function arguments, in `IF` / `WHILE` /
`LOOP ... WHERE` conditions, and after `RETURN`.

Calling
```
RESULT = THIS.CALCULATE(P1, P2);     /* same class or dialog */
RESULT = CUSTREC.CALCULATE(P1);      /* another instance */
RESULT = ABS(5 - 10);                /* system function */
RESULT = REC.CALCULATE(P1,,,P4);     /* P2, P3 take defaults */
```
Extra arguments are ignored and missing ones take the type default. Do not
rely on that for system functions.

## Implicit conversion

Simple types convert automatically on assignment (same rules for `MOVE` and
`=`); there is no cast operator. A failed conversion yields the destination's
default. TABLE, VECTOR and class instances are not convertible.

- STRING to number: parsed (dot as decimal separator); failure gives 0.
- STRING to DATE/DATETIME: parsed as `DD.MM.YYYY [HH:MM:SS]`; failure gives
  NULLDATE.
- Number to DATE/DATETIME: value is milliseconds since 01.01.1970.
- DECIMAL to INTEGER/LONG: whole part only.
- DATE/DATETIME to STRING: `DD.MM.YYYY` / `DD.MM.YYYY HH:mm:ss`.
- DATE/DATETIME to INTEGER/LONG: milliseconds since 01.01.1970.
- DATE/DATETIME to DECIMAL: not allowed, gives 0.
- DATETIME to DATE keeps the date part; DATE to DATETIME uses 00:00:00;
  DATETIME to TIME keeps the time part.

## Flow control

```
IF A > 1 && A < MAXVAL THEN
	RESULT = 'in range';
ELSE
	IF A <= 1 THEN
		RESULT = 'low';
	ELSE
		RESULT = 'high';
	ENDIF;
ENDIF;
```

```
SWITCH CODE
CASE 5:
	RESULT = 'five';
CASE '7','8':
	RESULT = 'seven or eight';
DEFAULT:
	RESULT = 'other';
ENDSWITCH;
```
The switch item must be a variable. Comparison is on string values. Several
values per `CASE` are comma separated. There is no fall-through and no `BREAK`.

```
WHILE N < 10
BEGIN
	N = N + 1;
ENDWHILE;
```

```
LOOP AT T1
BEGIN
	TOTAL = TOTAL + T1_AMOUNT;
ENDLOOP;
```

```
PARSE SOURCE INTO TOKEN DELIMITER '|'
BEGIN
	/* one iteration per token; default delimiter is newline */
ENDPARSE;
```

`BREAK;` and `CONTINUE;` work in `WHILE`, `LOOP` and `PARSE`.
`RETURN;` or `RETURN expression;` leaves the method.

## Strings

- No escape sequences. `TOCHAR(n)` returns the character for a decimal code:
  39 single quote, 10 newline.
- Concatenating with `+` converts numbers to text automatically.

| Function | Purpose |
| --- | --- |
| `STRLEN(s)` | length |
| `STRPOS(s, sub)` | position of `sub` in `s` |
| `STRSTR(s, index, length)` | substring |
| `STRLIKE()` | SQL LIKE-style pattern match |
| `ISNUMERIC(s)` | all characters numeric |
| `TRIM(s)` | strip leading/trailing whitespace |
| `REPLACE(s, old, new)` | replace |
| `LOWERCASE()`, `UPPERCASE()` | case conversion (language-aware) |
| `BASE64ENCODE(s, 'UTF-8')`, `BASE64DECODE(s, 'UTF-8')` | Base64 |
| `GETDIGEST(s, 'MD5')` | hash; also `'SHA1'` and others |

The public docs show `STRSTR` with a start index of both 0 and 1 in different
examples, and do not state what `STRPOS` returns when nothing is found. Confirm
index conventions against existing code or TROIA Help before relying on them.

String building
```
LOCAL:
	STRINGBUILDER SB;

APPENDSTRING 'Hello' TO SB;
APPENDSTRING ' World' TO SB;
```
Use STRINGBUILDER only for building; keep ordinary text in STRING.

## Dates and times

- All date types are stored as a long (milliseconds since 01.01.1970, subject
  to time zone). Subtracting two datetimes gives milliseconds.
- Read the current time from `SYS_CURRENTDATE`, not from a freshly defined
  variable's default. `CURRENTTIMEMILLIS()` returns it as a long and suits
  timing.
- Use the narrowest type: a date without time belongs in `DATE`.
- An empty date field holds NULLDATE; test with `NULLDATE(x)`.
- Use `SYS_MINDATE` / `SYS_MAXDATE` with `ISMINDATE()` / `ISMAXDATE()` instead
  of hard-coded sentinel dates.

| Function | Purpose |
| --- | --- |
| `DATETOMILLISECONDS(d)`, `MILLISECONDSTODATE(n)` | explicit long conversion |
| `ISDATE(x)` | 1 only when the variable's type is DATE or DATETIME |
| `CHECKDATE(s)`, `CHECKTIME(s)` | is the string a valid date / time |
| `GETDATE(d)`, `GETTIME(d)` | date part / time part |
| `GETDAY`, `GETMONTH`, `GETYEAR`, `GETHOUR`, `GETMINUTE`, `GETWEEK` | one component |
| `GETDAYOFWEEK(d)` | Monday = 1 |
| `GETDATEFROMWEEK(week, year)` | first day of that week |
| `FIRSTDATEINMONTH(month, year)`, `LASTDATEINMONTH(month, year)` | month bounds |
| `ADDDAYS`, `ADDYEARS`, `ADDHOURS`, `ADDMINUTES`, `SUBDAYS`, `SUBMONTHS`, ... | date arithmetic |
| `GETMINUTEDIFF(d1, d2)` | difference in minutes |
| `FORMATDATE(d, format)`, `PARSEDATE(s, format)` | explicit formats; 5.02.05+ |
| `GETDBDATESTR(d)` | date formatted for the database, for hand-built SQL |

Format strings are Java patterns (`'yyyy.MM.dd'`). The user's default formats
are in `SYS_DATEFORMAT`, `SYS_DATETIMEFORMAT`, `SYS_DATETIMESFORMAT`,
`SYS_TIMEFORMAT`, `SYS_TIMESFORMAT` (5.02.05+). Controls and table cells format
dates automatically; values are stored in the database time zone and shown in
the user's.

Other commands seen in the docs: `DELAY ms;` (sleep), `DUMP expr;` (write a
value to the trace).
