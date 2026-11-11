# LOML Specification

*Larry's Obvious, Minimal Language. Draft 0.1*

## 1. Overview

LOML is a human-readable, plain-text configuration format that maps unambiguously to hash tables and parses cleanly into native data structures. It is derived from TOML v1.0.0. Anything this specification does not mention behaves as in TOML v1.0.0, including TOML's rules for duplicate keys, implicitly created dictionaries, and redefinition.

- A LOML document is UTF-8 text. Lines end with LF or CRLF.
- Keys and values are case-sensitive.
- Control characters other than tab are not allowed (newlines excepted).
- TOML *tables* are called *dictionaries* in LOML. In LOML, a *table* is the row-and-column body type (section 6.5).
- MUST, MUST NOT, and SHOULD are used as in RFC 2119.

## 2. Lexical elements

### 2.1 Whitespace and comments

Whitespace is space or tab. Leading whitespace on a line is ignored, except inside multi-line strings and doc string content. A comment starts with `#` and runs to the end of the line. There are no block comments. A `#` inside a string or a doc string is content, not a comment.

### 2.2 Keys

- **Bare keys** contain only `A-Za-z0-9_-`.
- **Quoted keys** use a single-line basic (`"..."`) or literal (`'...'`) string. They may be empty or contain any characters, e.g. `"127.0.0.1" = "localhost"`.
- **Dotted keys** join keys with dots; whitespace around dots is ignored. `physical.color = "orange"` creates a `color` key inside a `physical` dictionary.

A key, the `=`, and its value start on the same line. A value spans several lines only through a multi-line string or a continuation (section 3.2).

## 3. Values

The value types are: string, integer, float, boolean, offset date-time, local date-time, local date, local time, array, and dictionary.

### 3.1 Strings

- **Basic strings** `"..."` support the escapes `\b \t \n \f \r \" \\ \uXXXX \UXXXXXXXX`.
- **Multi-line basic strings** `"""..."""` support the same escapes. A newline immediately after the opening delimiter is trimmed. A backslash at the end of a line trims the newline and all following whitespace, as in TOML.
- **Literal strings** `'...'` have no escapes.
- **Multi-line literal strings** `'''...'''` have no escapes. A newline immediately after the opening delimiter is trimmed.

### 3.2 String concatenation

Adjacent string literals are concatenated: `"a" "b" "c"` is `"abc"`, and `"quote:\n" '"raw\string"'` is the text `quote:`, a newline, then `"raw\string"`.

- Concatenation is allowed only where a value is not delimited by commas: after `=` in a dictionary body, after the marker in a multi-line array entry, and in table cells.
- It is **not** allowed inside inline arrays or inline dictionaries. `["a" "b"]` is an error, not `["ab"]`.
- Literals are separated by at least one space or tab, or by a continuation. All four string kinds may be mixed.
- **Continuation:** a backslash as the last character of a line (optionally preceded by whitespace, never followed by anything, not even a space or comment) continues the expression on the next line. Leading whitespace on that line is ignored. Without the backslash, the expression ends at the newline.
- Continuation is not allowed in table cells, because a table row is a single line.
- Concatenating a non-string is an error.

```loml
message = "Hello, " 'world' \
    "!"   # Hello, world!
```

### 3.3 Numbers

- **Integers:** optional sign, then decimal digits with no leading zeros. Underscores may separate digits (`1_000_000`). Prefixes `0x`, `0o`, `0b` give hex, octal, and binary (unsigned). The range is signed 64-bit; out of range is an error.
- **Floats:** IEEE 754 binary64. Require a decimal point with digits on both sides, an exponent, or both (`3.14`, `5e+22`, `-2.5E-3`). The special values `inf`, `+inf`, `-inf`, `nan`, `+nan`, `-nan` are valid in lowercase.

### 3.4 Booleans

Lowercase `true` or `false`.

### 3.5 Dates and times

RFC 3339 formats: offset date-time (`1979-05-27T07:32:00Z`, `1979-05-27 07:32:00-07:00`), local date-time (`1979-05-27T07:32:00`), local date (`1979-05-27`), and local time (`07:32:00`). The date-time separator is `T`, `t`, or a single space.

### 3.6 Inline arrays

`[1, 2, 3]`. An inline array is on one line, items are comma-separated, trailing commas are prohibited, and types may be mixed. Whitespace around items is ignored. `[]` is the way to write an empty array.

### 3.7 Inline dictionaries

`point = { x = 1, y = 2 }`. The same rules as inline arrays: one line, no trailing comma. `{}` is the empty dictionary.

## 4. Document structure

A document is a **root body** followed by zero or more **sections**.

The type of a body is decided by its first non-blank, non-comment line:

- a key: a **dictionary** body
- ` -  ` : a **multi-line array** body
- `>` : a **doc string** body
- `|` : a **table** body
- no such line: an empty **dictionary**

A body has exactly one type. Mixing line kinds in one body is an error.

If the root body is a dictionary (including empty), any sections may follow. If the root body is an array, doc string, or table, the document itself is that value and no header may follow it.

## 5. Sections

A header is `[path]` or `[[path]]`, where *path* is a (possibly dotted) key. Its body runs until the next header or end of input.

- `[path]` sets the value at *path* to the body's value. A dictionary body is the usual TOML table.
- `[[path]]` appends the body's value as a new element of the array at *path*, creating the array if needed. With a dictionary body this is TOML's array of tables. With a table body it yields an array of arrays of dictionaries. Elements may have different body types.
- An empty body is an empty dictionary.
- A header whose body is not a dictionary defines a leaf. No `[path.child]` or `[[path.child]]` header may appear anywhere in the document, before or after it.
- A header whose path passes through a `[[x]]` array refers to the most recent element of `x`, which must be a dictionary.

All other TOML redefinition rules apply (section 7).

## 6. Body types

In every body type except doc strings, blank lines and comment lines may appear between entries, and a comment may follow an entry on its line.

### 6.1 Dictionary

Lines of `key = value`, with the semantics of TOML key/value pairs. Only dictionary bodies may contain `key = value` lines.

### 6.2 Multi-line array

Each entry is a line starting with a marker, a hyphen followed by one or more spaces or tabs, then a value (which may use concatenation).

```loml
[count_to_three]
- 1
- 2
- 3
```

The space after the hyphen distinguishes entries from negative numbers: `- 1` is an entry with positive 1, and `- -1` is an entry with negative one. An empty array cannot be written as a body; use `key = []` in the parent dictionary.

### 6.3 Multi-dimensional array

The first entry carries *k* markers (`- - 1`), which fixes the array's depth *k*. Indentation is ignored.

Let level 1 be the body array itself, nested down to level *k*, the innermost array that holds values. A line with *m* markers (1 <= *m* <= *k*) starts a fresh array at each of levels *k-m+2* through *k*, then appends its value to the level-*k* array. So a line with a single marker appends to the current innermost array, and a line with *k* markers starts a new element of level 1's array.

- A line with more than *k* markers is an error.
- Ragged arrays are allowed.
- Depth counts markers only; an inline array value such as `- [1, 2]` is just a value.

```loml
[grid]
- - 1
  - 2
- - 3
  - 4
```

This is `[[1, 2], [3, 4]]`.

### 6.4 Doc strings

Each line is ` >  ` followed by text, or just `>` for an empty line.

- The `>` and exactly one following space are removed. All other characters, including further spaces, `#`, quotes, and backslashes, are kept literally. There are no escapes.
- Lines are joined with LF. The newline after the last line is not included. To end the string with a newline, add a final `>` line.
- Lines must be contiguous. A blank line or comment line ends the doc string.
- Parsers MUST accept ` >  ` with a trailing space as an empty line. Writers SHOULD emit `>`, since editors often strip trailing whitespace.
- A doc string is a body, never a value after `=`. For a multi-line string inside a dictionary, use `"""` or `'''`.

```loml
[a]
> hello

[b]
> hello
>
```

Here `a` is `hello` and `b` is `hello` followed by a newline.

### 6.5 Tables

Each row starts and ends with `|`. Each cell is a value, with the syntax used after `=` (concatenation allowed, continuation not).

- A header row is present only if a separator row follows it. The separator row has the same number of cells, each with at least one hyphen: `| --- | --- |`. Header cells follow key syntax, including dotted keys.
- With a header, the table is an array of dictionaries. A dotted header `a.b` creates a nested dictionary in every row.
- Without a header, the table is a two-dimensional array.
- Every row MUST have the same number of cells as the header, or as the first row if there is no header.
- A header with zero data rows is an error. Duplicate or conflicting headers (such as `a` and `a.b`) are errors.

```loml
[tasks]
| task | owner.name | done |
| ---- | ---------- | ---- |
| "Fix login" | "Alex" | true |
| "Add dark mode" | "Jordan" | false |

[matrix]
| 1 | 2 |
| 3 | 4 |
```

The same data model results from `[[tasks]]` sections with dictionary bodies.

## 7. Errors

A conforming parser MUST reject a document that:

- defines a key twice, conflicts through dotted keys, or redefines a section or an inline dictionary, per TOML v1.0.0
- mixes body types, or has a header after a non-dictionary root body
- has a sub-header under a section with a non-dictionary body
- has a multi-dimensional entry with more markers than the first entry
- has a table row with the wrong cell count, a header with no rows, or duplicate or conflicting headers
- has a trailing comma or a newline inside an inline array or dictionary
- concatenates a non-string, or concatenates inside an inline array or dictionary
- has a doc string line after a blank or comment line within the same body
- contains an invalid escape, invalid date or time, or out-of-range integer

## 8. ABNF

The grammar uses RFC 5234 with the core rules ALPHA, DIGIT, HEXDIG, and DQUOTE. Where alternatives share a prefix (such as three quotes versus an empty string), the longest match wins. A final line may omit its newline; treat end of input as a newline. Constraints that ABNF cannot express (marker counts, cell counts, duplicate keys) are in sections 5 to 7.

```abnf
; ---------------------------------------------------------------
; Document
; ---------------------------------------------------------------
document        = root-other / ( dict-body *section )
root-other      = array-body / docstring-body / table-body

section         = header-line section-body
header-line     = std-header / array-header
std-header      = ws "[" ws key ws "]" line-end
array-header    = ws "[[" ws key ws "]]" line-end
section-body    = array-body / docstring-body / table-body / dict-body

; ---------------------------------------------------------------
; Whitespace, comments, lines
; ---------------------------------------------------------------
ws              = *wschar
wschar          = %x20 / %x09
newline         = %x0A / ( %x0D %x0A )
comment         = "#" *non-eol
non-eol         = %x09 / %x20-7E / non-ascii
non-ascii       = %x80-D7FF / %xE000-10FFFF
line-end        = ws [ comment ] newline
blank-line      = line-end

; ---------------------------------------------------------------
; Bodies
; ---------------------------------------------------------------
dict-body       = *( blank-line / keyval-line )
keyval-line     = ws keyval line-end
keyval          = key ws "=" ws value

array-body      = *blank-line array-line *( blank-line / array-line )
array-line      = ws 1*marker value line-end
marker          = "-" 1*wschar

docstring-body  = *blank-line docstring-line *docstring-line *blank-line
docstring-line  = ws ">" [ %x20 docstring-text ] newline
docstring-text  = *( %x09 / %x20-7E / non-ascii )

table-body      = *blank-line ( table-with-header / table-no-header )
table-with-header = header-row separator-row
                    *blank-line table-row *( blank-line / table-row )
table-no-header = table-row *( blank-line / table-row )
header-row      = ws "|" 1*( ws key ws "|" ) line-end
separator-row   = ws "|" 1*( ws 1*"-" ws "|" ) line-end
table-row       = ws "|" 1*( ws cell ws "|" ) line-end
cell            = cell-string / non-string-value
cell-string     = string *( 1*wschar string )

; ---------------------------------------------------------------
; Keys
; ---------------------------------------------------------------
key             = dotted-key / simple-key
simple-key      = quoted-key / unquoted-key
unquoted-key    = 1*( ALPHA / DIGIT / "-" / "_" )
quoted-key      = basic-string / literal-string
dotted-key      = simple-key 1*( ws "." ws simple-key )

; ---------------------------------------------------------------
; Values
; ---------------------------------------------------------------
value           = string-concat / non-string-value
inline-value    = string / non-string-value
non-string-value = boolean / array / inline-dict / date-time / float / integer

string-concat   = string *( concat-sep string )
concat-sep      = 1*wschar / ( ws %x5C newline ws )

array           = "[" ws [ inline-value *( ws "," ws inline-value ) ] ws "]"
inline-dict     = "{" ws [ inline-keyval *( ws "," ws inline-keyval ) ] ws "}"
inline-keyval   = key ws "=" ws inline-value

; ---------------------------------------------------------------
; Strings
; ---------------------------------------------------------------
string          = ml-basic-string / basic-string /
                  ml-literal-string / literal-string

basic-string    = DQUOTE *basic-char DQUOTE
basic-char      = basic-unescaped / escaped
basic-unescaped = wschar / %x21 / %x23-5B / %x5D-7E / non-ascii
escaped         = %x5C ( %x22 / %x5C / %x62 / %x66 / %x6E / %x72 / %x74 /
                         ( %x75 4HEXDIG ) / ( %x55 8HEXDIG ) )

ml-basic-string = 3DQUOTE [ newline ] ml-basic-body 3DQUOTE
ml-basic-body   = *mlb-content *( mlb-quotes 1*mlb-content ) [ mlb-quotes ]
mlb-content     = basic-char / newline / mlb-continuation
mlb-quotes      = 1*2DQUOTE
mlb-continuation = %x5C ws newline *( wschar / newline )

literal-string  = apostrophe *literal-char apostrophe
literal-char    = %x09 / %x20-26 / %x28-7E / non-ascii
apostrophe      = %x27

ml-literal-string = 3apostrophe [ newline ] ml-literal-body 3apostrophe
ml-literal-body = *mll-content *( mll-quotes 1*mll-content ) [ mll-quotes ]
mll-content     = literal-char / newline
mll-quotes      = 1*2apostrophe

; ---------------------------------------------------------------
; Numbers and booleans
; ---------------------------------------------------------------
integer         = hex-int / oct-int / bin-int / dec-int
dec-int         = [ "-" / "+" ] unsigned-dec-int
unsigned-dec-int = DIGIT / ( %x31-39 1*( DIGIT / ( "_" DIGIT ) ) )
hex-int         = %x30.78 HEXDIG *( HEXDIG / ( "_" HEXDIG ) )
oct-int         = %x30.6F %x30-37 *( %x30-37 / ( "_" %x30-37 ) )
bin-int         = %x30.62 %x30-31 *( %x30-31 / ( "_" %x30-31 ) )

float           = special-float / ( dec-int ( exp / ( frac [ exp ] ) ) )
frac            = "." zero-prefixable-int
exp             = "e" [ "-" / "+" ] zero-prefixable-int
zero-prefixable-int = DIGIT *( DIGIT / ( "_" DIGIT ) )
special-float   = [ "-" / "+" ] ( %x69.6E.66 / %x6E.61.6E )

boolean         = %x74.72.75.65 / %x66.61.6C.73.65

; ---------------------------------------------------------------
; Dates and times (RFC 3339)
; ---------------------------------------------------------------
date-time       = offset-date-time / local-date-time / local-date / local-time
offset-date-time = full-date time-delim full-time
local-date-time = full-date time-delim partial-time
local-date      = full-date
local-time      = partial-time

full-date       = 4DIGIT "-" 2DIGIT "-" 2DIGIT
partial-time    = 2DIGIT ":" 2DIGIT ":" 2DIGIT [ "." 1*DIGIT ]
full-time       = partial-time time-offset
time-offset     = "Z" / ( ( "+" / "-" ) 2DIGIT ":" 2DIGIT )
time-delim      = "T" / %x20
```

In the ABNF, quoted letters such as `"T"`, `"Z"`, and `"e"` match either case, as RFC 5234 defines. `inf`, `nan`, `true`, `false`, and the `0x`, `0o`, `0b` prefixes use explicit hex terminals so that only lowercase matches.

## 9. Example

```loml
# root dictionary
title = "LOML Example"
owner.name = "Larry"
ports = [8000, 8001]
motto = "Obvious, " "minimal."

[owners]
| name | role.title | since |
| ---- | ---------- | ----- |
| "Ann" | "admin" | 2020-01-02 |
| "Bo" | "dev" | 2021-03-04 |

[[steps]]
> echo one

[[steps]]
- 1
- 2
```

Invalid documents:

```loml
# trailing comma in inline array
bad = [1, 2,]

# concatenation inside an array
bad = ["a" "b"]

# mixed body types
[x]
key = 1
- 2

# sub-header under a non-dictionary section
[x]
- 1
[x.y]
key = 1
```

## 10. Decisions made beyond the original answers

These points were not settled explicitly and are filled in above. Change any that you disagree with.

- Ragged multi-dimensional arrays are allowed.
- An empty section body is an empty dictionary; empty arrays, strings, and tables are written as values (`[]`, `""`) in a parent dictionary.
- Blank and comment lines are allowed between entries of arrays and tables, but a blank or comment line ends a doc string.
- Table cells allow string concatenation (on one line), since `|` already delimits them.
- Writers emit `>` for empty doc string lines; parsers accept ` >  ` too.
- A marker is a hyphen followed by one or more spaces or tabs.
- A non-dictionary section is a leaf: no sub-headers before or after it.
- Baseline is TOML v1.0.0, so mixed-type arrays, lowercase `t` and `z`, and a space date-time separator are allowed, and inline dictionaries and arrays are single-line.
